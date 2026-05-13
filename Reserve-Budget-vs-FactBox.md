# เปรียบเทียบ Logic: ปุ่ม Reserve Budget vs. FactBox Available Budget

> **เอกสารสำหรับทีม BA + Developer**
> ขอบเขต: หน้า WEH_Purch. Quote - Corporate (Page 50094)
> วันที่: 2026-05-13
> Object หลักที่เกี่ยวข้อง: `Cod50009` (Logic), `Pag50104` (FactBox), `Pag50094` (UI)

---

## สารบัญ

1. [สรุปสั้น](#1-สรุปสั้น)
2. [Table & Field Dictionary](#2-table--field-dictionary)
3. [Logic การเช็ค Pass / Not Pass](#3-logic-การเช็ค-pass--not-pass)
4. [Logic FactBox Available Budget](#4-logic-factbox-available-budget)
5. [จุดต่างที่ทำให้ตัวเลขไม่ตรงกัน](#5-จุดต่างที่ทำให้ตัวเลขไม่ตรงกัน)
6. [ข้อเสนอแนะ](#6-ข้อเสนอแนะ)
7. [Appendix: Code Reference](#7-appendix-code-reference)

---

## 1. สรุปสั้น

- **ปุ่ม Reserve Budget** เช็คผ่าน/ไม่ผ่าน → ทำที่ `Cod50009.CheckAvailableBudget`
- **FactBox โชว์ Available** → ทำที่ `Pag50104.GetBudgetInfo`
- ทั้งสองอ่าน Table เดียวกัน **แต่ใช้ filter ต่างกัน** → ตัวเลขอาจไม่ตรงกัน
- **FactBox = ดูประมาณการ** ไม่ใช่ตัวตัดสิน
- **CheckAvailableBudget = ตัวจริงที่ตัดสิน Pass / Not Pass** เข้มกว่า ละเอียดกว่า

---

## 2. Table & Field Dictionary

### 2.1 Table ที่ใช้

| Table | บทบาท |
|-------|------|
| `Purchase Header` (38) | หัวเอกสาร PR/PO/Invoice |
| `Purchase Line` (39) | บรรทัดเอกสาร |
| `G/L Budget Entry` (96) | งบประมาณ (ทั้ง Budget ตั้งต้นและ Encumbrance) |
| `G/L Budget Name` (95) | ชื่อ Budget |
| `G/L Entry` (17) | รายการที่โพสจริง |
| `General Ledger Setup` (98) | Setup ระดับองค์กร |
| `WEH_G/L Budget Enc. PR/PO` (50007) | Encumbrance Ledger (PR/PO/PI sub-entries) |

### 2.2 Field สำคัญใน `Purchase Line`

| Field | ความหมาย |
|-------|---------|
| `WEH_Budget Type` | C = งบกลาง, O = งบโครงการ, P = ไม่เช็ค |
| `WEH_Budget Name` | ชื่อ Budget ที่ Line ผูกอยู่ |
| `WEH_Check Budget` | สถานะ: `' '` (ว่าง) / `Pass` / `Not Pass` |
| `RBG_Ref. G/L (Item/FA)-Budget` | G/L Account สำหรับเทียบงบ (กรณี Line เป็น Item / FA) |
| `Type` | G/L Account / Item / Fixed Asset / Resource / Comment |
| `No.` | รหัสตาม Type |
| `Shortcut Dimension 1 Code` | Dim 1 (ส่วนใหญ่ = แผนก / โครงการ) |
| `Shortcut Dimension 2 Code` | Dim 2 (ส่วนใหญ่ = ฝ่าย / Cost Center) |
| `Outstanding Amt. Ex. VAT (LCY)` | ยอดคงเหลือไม่รวม VAT |
| `Line Amount` | ยอดบรรทัด |

### 2.3 Field สำคัญใน `G/L Budget Entry`

| Field | ความหมาย |
|-------|---------|
| `Budget Name` | ชื่อ Budget (ใช้แยก Budget ตั้งต้น vs. Encumbrance) |
| `G/L Account No.` | บัญชี G/L ที่ผูก |
| `Amount` | จำนวนเงิน Budget |
| `Global Dimension 1 Code` | Dim 1 |
| `Global Dimension 2 Code` | Dim 2 |
| `WEH_Budget Type` | C / O / P |
| `WEH_Encumbrance PR Amount` *(FlowField)* | ยอดจอง PR (Purchase Requisition) |
| `WEH_Encumbrance PO Amount` *(FlowField)* | ยอดจอง PO (Purchase Order) |
| `WEH_Encumbrance PI Amount` *(FlowField)* | ยอดจอง PI (Purchase Invoice) |
| `Encumbrance PCN Amt.` *(FlowField)* | ยอดจาก Posted Credit Memo |

### 2.4 Field สำคัญใน `G/L Budget Name`

| Field | ความหมาย |
|-------|---------|
| `Name` | ชื่อ Budget |
| `WEH_Encumbrance Name` | ชื่อ Budget Encumbrance ที่ผูกคู่กัน |
| `WEH_Start Date` | วันเริ่มงบ |
| `WEH_End Date` | วันสิ้นสุดงบ |

### 2.5 Field สำคัญใน `General Ledger Setup`

| Field | ความหมาย |
|-------|---------|
| `WEH_G/L for Budget Type C` | G/L Account กลางที่ใช้กับ Budget Type C |

### 2.6 Field สำคัญใน `G/L Account`

| Field | ความหมาย |
|-------|---------|
| `WHE_SkipCheckBudget` | ถ้า = true → ข้ามบัญชีนี้ตอนคำนวณ Actual (Type C) |

---

## 3. Logic การเช็ค Pass / Not Pass

**Procedure**: `CheckAvailableBudget` ใน `Cod50009.WEHBudgetControlManagement.al` (บรรทัด 1170-1414)

**สูตรหลัก**:
```
Available = Budget − Encumbrance − Actual
PRInLineAmt = Σ ยอดของ Line ในเอกสารเดียวกันที่ Filter ตรงกัน

ถ้า PRInLineAmt <= Available  → Line.WEH_Check Budget := Pass
มิเช่นนั้น                    → Line.WEH_Check Budget := Not Pass
```

### 3.1 Type O (งบโครงการ)

**G/L Account ที่ใช้ (`TEMP_ACC_NO`)** — มี if/else:
```al
if PurchLine.Type = PurchLine.Type::"G/L Account" then
    TEMP_ACC_NO := PurchLine."No."
else
    TEMP_ACC_NO := PurchLine."RBG_Ref. G/L (Item/FA)-Budget";
```

| รายการ | Source Table | Filter ที่ใช้ |
|--------|--------------|-------------|
| **Budget** | `G/L Budget Entry` | `Budget Name = Line."WEH_Budget Name"`<br>`G/L Account No. = TEMP_ACC_NO`<br>`Global Dimension 1 Code = Line."Shortcut Dimension 1 Code"`<br>`WEH_Budget Type = O` |
| **Encumbrance** | `G/L Budget Entry` (Encumbrance) | `Budget Name = GLBudgetName."WEH_Encumbrance Name"`<br>`G/L Account No. = TEMP_ACC_NO`<br>`Global Dimension 1 Code = Line.Dim1`<br>`WEH_Budget Type = O`<br>→ Σ (`PR Amount` + `PO Amount` + `PI Amount`) |
| **Actual** | `G/L Entry` | `G/L Account No. = Line."RBG_Ref. G/L (Item/FA)-Budget"`<br>`Global Dimension 1 Code = Line.Dim1`<br>`Posting Date ∈ [Start, End]` |

### 3.2 Type C (งบกลาง)

| รายการ | Source Table | Filter ที่ใช้ |
|--------|--------------|-------------|
| **Budget** | `G/L Budget Entry` | `Budget Name = Line."WEH_Budget Name"`<br>`G/L Account No. = GLSetup."WEH_G/L for Budget Type C"`<br>`Global Dimension 2 Code = Line."Shortcut Dimension 2 Code"` |
| **Encumbrance** | `G/L Budget Entry` (Encumbrance) | `Budget Name = WEH_Encumbrance Name`<br>`Global Dimension 1 Code = Line.Dim1`<br>`Global Dimension 2 Code = Line.Dim2`<br>`WEH_Budget Type = C`<br>→ Σ (`PR Amount` + `PO Amount` + `PI Amount`) |
| **Actual** | `G/L Entry` | `Global Dimension 1 Code = Line.Dim1`<br>`Global Dimension 2 Code = Line.Dim2`<br>`Posting Date ∈ [Start, End]`<br>**ข้ามทุกบรรทัดที่ `G/L Account.WHE_SkipCheckBudget = true`** |

### 3.3 Type P
→ **ผ่านเลย** ไม่ตรวจสอบใด ๆ

### 3.4 ยอดที่นำมาเทียบ (`PRInLineAmt`)

```
PRInLineAmt = Σ Cal_AmountPurchaseLine(Line ทุกบรรทัดในเอกสารเดียวกัน
                                       ที่ Dim + Budget Type + Budget Name
                                       ตรงกัน)
```

จากนั้น:
```
IF PRInLineAmt <= AvaiableBudget THEN LinePass := TRUE
```

---

## 4. Logic FactBox Available Budget

**Page**: `Pag50104.WEHAvailableBudgetFactbox.al`
**Procedure**: `GetBudgetInfo` (บรรทัด 66-177)
**Trigger**: `OnAfterGetRecord` + `OnAfterGetCurrRecord` → คำนวณใหม่ทุกครั้งที่เปลี่ยน Line

**สูตร**:
```
Available = Budget − PR − PO − Actual
```

โดยคำนวณจาก **Line ปัจจุบันที่ผู้ใช้คลิกอยู่** (Rec)

### 4.1 Type O

| ช่อง | Source Table | Filter |
|------|--------------|--------|
| **Budget** | `G/L Budget Entry` | `Budget Name = Rec."WEH_Budget Name"`<br>`G/L Account No. = Rec."RBG_Ref. G/L (Item/FA)-Budget"` *(ไม่ดู Line.Type)*<br>`Global Dimension 1 Code = Rec.Dim1`<br>`WEH_Budget Type = O` |
| **Reserve Budget (PR)** | `G/L Budget Entry` (Encumbrance) | Filter เหมือน Budget แต่ใช้ `WEH_Encumbrance Name`<br>→ Σ `WEH_Encumbrance PR Amount` |
| **Obligate Budget (PO)** | เหมือน PR | → Σ `WEH_Encumbrance PO Amount` |
| **Actual** | เหมือน PR | → Σ (`WEH_Encumbrance PI Amount` + `Encumbrance PCN Amt.`) |

### 4.2 Type C

| ช่อง | Source Table | Filter |
|------|--------------|--------|
| **Budget** | `G/L Budget Entry` | `Budget Name = Rec."WEH_Budget Name"`<br>**`Global Dimension 2 Code = Rec.Dim2` อย่างเดียว**<br>`WEH_Budget Type = C`<br>(**ไม่ filter G/L Account No.**) |
| **PR / PO / Actual** | `G/L Budget Entry` (Encumbrance) | Filter Encumbrance Name + Dim2 + Type=C<br>(**ไม่ filter Dim1, ไม่ filter G/L Account**) |

### 4.3 Type P
→ **ไม่มี logic** Budget ค่า 0 หมด

---

## 5. จุดต่างที่ทำให้ตัวเลขไม่ตรงกัน

| # | เรื่อง | ตัวเช็ค Pass (`CheckAvailableBudget`) | FactBox (`GetBudgetInfo`) | Code Reference |
|---|------|--------------------------------------|--------------------------|----------------|
| 1 | **Actual มาจากไหน** | `G/L Entry.Amount` (โพสจริง) | `WEH_Encumbrance PI Amount` + `Encumbrance PCN Amt.` (จาก Encumbrance) | Cod50009 บรรทัด 1296-1306 vs Pag50104 บรรทัด 118 |
| 2 | **Type O: G/L Account** | `TEMP_ACC_NO` (ขึ้นกับ `Line.Type`) | `Rec."RBG_Ref. G/L"` เสมอ | Cod50009 บรรทัด 1261-1267 vs Pag50104 บรรทัด 97 |
| 3 | **Type C: Budget filter** | filter G/L = `GLSetup."WEH_G/L for Budget Type C"` | ไม่ filter G/L Account | Cod50009 บรรทัด 1333-1334 vs Pag50104 บรรทัด 140-142 |
| 4 | **Type C: Encumbrance filter** | Dim1 + Dim2 ทั้งสอง | Dim2 เท่านั้น | Cod50009 บรรทัด 1345-1347 vs Pag50104 บรรทัด 151-153 |
| 5 | **Type C: Skip บัญชี** | ข้าม G/L ที่ `WHE_SkipCheckBudget = true` | ไม่มี logic นี้ | Cod50009 บรรทัด 1364-1370 |
| 6 | **ระดับการรวม** | รวมทุก Line ของเอกสารแล้วเทียบ | โชว์ pool เฉย ๆ ไม่บอกว่าใช้ไปเท่าไหร่ | Cod50009 บรรทัด 1377-1398 |
| 7 | **PCN (Posted Credit Memo)** | ไม่นับ (PR+PO+PI เท่านั้น) | นับใน Actual | Cod50009 บรรทัด 1290-1292 vs Pag50104 บรรทัด 118 |

---

## 6. ข้อเสนอแนะ

1. **อย่าใช้ตัวเลขใน FactBox ตัดสินใจว่าจะผ่านหรือไม่** — ให้กดปุ่ม Reserve Budget แล้วดูผลจริง
2. ถ้าผู้ใช้ถามว่า *"ทำไม FactBox โชว์ว่ามีเงิน แต่ Reserve ไม่ผ่าน"* → คำตอบมักเป็นข้อ 1, 4, 5 ในตารางข้างบน
3. ถ้าต้องการให้ FactBox ตรงกับ logic เช็คจริง → ต้องปรับ `Pag50104.GetBudgetInfo` ให้ filter เหมือน `CheckAvailableBudget` (เป็น scope งานแยก ต้องสรุป requirement ก่อน)
4. แนะนำให้เพิ่ม Tooltip / Help text บน FactBox ว่า *"ตัวเลขนี้เป็นค่าประมาณการเท่านั้น กรุณากด Reserve Budget เพื่อตรวจสอบจริง"*

---

## 7. Appendix: Code Reference

### 7.1 Entry Point: ปุ่ม Reserve Budget

📍 `3 - Page/Pag50094.WEHPurchQuoteCorporate.al` บรรทัด 1398-1432

```al
action("WEH_Check Available Budget")
{
    ApplicationArea = All;
    Caption = 'Reserve Budget';
    Image = CheckList;
    Enabled = ReserveEnabled;

    trigger OnAction()
    var
        BudgetControlMgt: Codeunit "WEH_Budget Control Management";
        Purchase_Line_Table: Record "Purchase Line";
    begin
        // ตรวจ Line ทุกบรรทัดมี Quantity > 0
        Purchase_Line_Table.Reset();
        Purchase_Line_Table.SetRange("Document No.", Rec."No.");
        Purchase_Line_Table.SetRange("Document Type", Rec."Document Type");
        Purchase_Line_Table.SetFilter(Type, '<>%1', "Purchase Line Type"::" ");
        if Purchase_Line_Table.FindSet() then begin
            repeat
                if Purchase_Line_Table.Quantity = 0 then
                    Error('There are still Line No.' + Format(Purchase_Line_Table."Line No.") +
                          ' with quantity greater than 0, please check before reserving the budget.');
            until Purchase_Line_Table.Next() = 0;
        end else
            Error('There is no valid purchase line to reserve budget.');

        if Confirm('Do you want to Reserve Budget in PR No. ' + Rec."No.") then begin
            CLEAR(BudgetControlMgt);
            BudgetControlMgt.CheckBudget(Rec);   // ← เรียก Logic หลัก
        end;
    end;
}
```

### 7.2 Entry ของ Codeunit

📍 `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` บรรทัด 1011-1043

```al
procedure CheckBudget(VAR PurchaseHead: Record "Purchase Header")
begin
    PurchaseHead.TESTFIELD("Document Date");

    // ถ้าเช็คผ่านไปแล้ว → exit
    IF PurchaseHead."WEH_Check Budget" = PurchaseHead."WEH_Check Budget"::PASS THEN
        exit;

    CheckPurchLineInit(PurchaseHead."Document Type", PurchaseHead."No.");

    if CheckBudgetByLine(PurchaseHead."Document Type", PurchaseHead."No.") then
        PurchaseHead."WEH_Check Budget" := PurchaseHead."WEH_Check Budget"::PASS
    else
        PurchaseHead."WEH_Check Budget" := PurchaseHead."WEH_Check Budget"::"Not Pass";

    PurchaseHead.Modify();
    if not PurchaseHead."Do Final PO" then
        MESSAGE('Reserve Budget Complete');
end;
```

### 7.3 จุดสำคัญใน `CheckAvailableBudget`

📍 `Cod50009` บรรทัด 1170-1414 (ตัดมาเฉพาะส่วนคำนวณ)

#### Type O — กำหนด G/L Account
```al
// บรรทัด 1261-1267
if PurchLine.Type = PurchLine.Type::"G/L Account" then
    TEMP_ACC_NO := PurchLine."No."
else
    TEMP_ACC_NO := PurchLine."RBG_Ref. G/L (Item/FA)-Budget";

// บรรทัด 1269-1278 — ดึง Budget
GLBudgetEntry.SETRANGE("Budget Name", PurchLine."WEH_Budget Name");
GLBudgetEntry.SETRANGE("G/L Account No.", TEMP_ACC_NO);
GLBudgetEntry.SETRANGE("Global Dimension 1 Code", PurchLine."Shortcut Dimension 1 Code");
GLBudgetEntry.SETRANGE("WEH_Budget Type", PurchLine."WEH_Budget Type");
IF GLBudgetEntry.FINDSET() THEN
    REPEAT
        BudgetAmt += GLBudgetEntry.Amount;
    UNTIL GLBudgetEntry.NEXT() = 0;

// บรรทัด 1287-1294 — ดึง Encumbrance
REPEAT
    GLBudgetEntry.CALCFIELDS("WEH_Encumbrance PR Amount", "WEH_Encumbrance PO Amount");
    EncumbranceAmt += GLBudgetEntry."WEH_Encumbrance PR Amount";
    EncumbranceAmt += GLBudgetEntry."WEH_Encumbrance PO Amount";
    EncumbranceAmt += GLBudgetEntry."WEH_Encumbrance PI Amount";
UNTIL GLBudgetEntry.NEXT = 0;

// บรรทัด 1296-1306 — ดึง Actual (จาก G/L Entry จริง!)
GLEntry.SETRANGE("G/L Account No.", PurchLine."RBG_Ref. G/L (Item/FA)-Budget");
GLEntry.SETRANGE("Posting Date", GLBudgetName."WEH_Start Date", GLBudgetName."WEH_End Date");
GLEntry.SETRANGE("Global Dimension 1 Code", PurchLine."Shortcut Dimension 1 Code");
IF GLEntry.FINDSET() THEN
    REPEAT
        ActualAmt := ActualAmt + GLEntry.Amount;
    UNTIL GLEntry.NEXT() = 0;

AvaiableBudget := BudgetAmt - EncumbranceAmt - ActualAmt;
```

#### Type C — Skip บัญชีบางตัว
```al
// บรรทัด 1363-1372 — Actual สำหรับ Type C
IF GLEntry.FINDSET() THEN
    REPEAT
        if GL_Account_Table.Get(GLEntry."G/L Account No.") then begin
            if GL_Account_Table.WHE_SkipCheckBudget then begin
                // ข้าม
            end else begin
                ActualAmt += GLEntry.Amount;
            end;
        end;
    UNTIL GLEntry.NEXT() = 0;
```

#### การเทียบ Pass / Not Pass
```al
// บรรทัด 1400-1411
IF PRInLineAmt <= AvaiableBudget THEN
    LinePass := TRUE;
...
if LinePass then
    PurchLine."WEH_Check Budget" := PurchLine."WEH_Check Budget"::Pass
else
    PurchLine."WEH_Check Budget" := PurchLine."WEH_Check Budget"::"Not Pass";
PurchLine.MODIFY();
```

### 7.4 จุดสำคัญใน FactBox `GetBudgetInfo`

📍 `3 - Page/Pag50104.WEHAvailableBudgetFactbox.al` บรรทัด 66-177

#### Type O
```al
// บรรทัด 92-104 — Budget
IF Rec."WEH_Budget Type" = Rec."WEH_Budget Type"::O THEN BEGIN
    GLBudgetEntry.SETRANGE("Budget Name", Rec."WEH_Budget Name");
    GLBudgetEntry.SetRange("G/L Account No.", Rec."RBG_Ref. G/L (Item/FA)-Budget");
    GLBudgetEntry.SETRANGE("Global Dimension 1 Code", Rec."Shortcut Dimension 1 Code");
    GLBudgetEntry.SETRANGE("WEH_Budget Type", Rec."WEH_Budget Type");
    IF GLBudgetEntry.FINDSET() THEN
        REPEAT
            Out_Budget_Amount := Out_Budget_Amount + GLBudgetEntry.Amount;
        UNTIL GLBudgetEntry.NEXT() = 0;
END;

// บรรทัด 113-121 — PR / PO / Actual (จาก Encumbrance ทั้งหมด)
IF GLBudgetEntry.FindSet() THEN
    REPEAT
        GLBudgetEntry.CALCFIELDS("WEH_Encumbrance PR Amount",
                                 "WEH_Encumbrance PO Amount",
                                 "Encumbrance PCN Amt.");
        Out_PR_Amount     += GLBudgetEntry."WEH_Encumbrance PR Amount";
        Out_PO_Amount     += GLBudgetEntry."WEH_Encumbrance PO Amount";
        Out_Actual_Amount += (GLBudgetEntry."WEH_Encumbrance PI Amount" +
                              GLBudgetEntry."Encumbrance PCN Amt.");
    UNTIL GLBudgetEntry.NEXT() = 0;

Out_Avaiable_Amount := Out_Budget_Amount - Out_PR_Amount
                       - Out_PO_Amount - Out_Actual_Amount;
```

#### Type C
```al
// บรรทัด 137-147 — Budget (สังเกตว่าไม่มี G/L Account filter)
IF Rec."WEH_Budget Type" = Rec."WEH_Budget Type"::C THEN BEGIN
    GLBudgetEntry.SETRANGE("Budget Name", Rec."WEH_Budget Name");
    GLBudgetEntry.SETRANGE("Global Dimension 2 Code", Rec."Shortcut Dimension 2 Code");
    GLBudgetEntry.SETRANGE("WEH_Budget Type", Rec."WEH_Budget Type");
    IF GLBudgetEntry.FindSet() THEN
        REPEAT
            Out_Budget_Amount += GLBudgetEntry.Amount;
        UNTIL GLBudgetEntry.NEXT() = 0;

// บรรทัด 149-161 — Encumbrance (ก็ไม่มี G/L Account filter, ไม่มี Dim1)
    GLBudgetEntry.SETRANGE("Budget Name", GLBudgetName."WEH_Encumbrance Name");
    GLBudgetEntry.SETRANGE("Global Dimension 2 Code", Rec."Shortcut Dimension 2 Code");
    GLBudgetEntry.SETRANGE("WEH_Budget Type", Rec."WEH_Budget Type");
    IF GLBudgetEntry.FINDSET() THEN
        REPEAT
            GLBudgetEntry.CALCFIELDS("WEH_Encumbrance PR Amount",
                                     "WEH_Encumbrance PO Amount",
                                     "Encumbrance PCN Amt.");
            Out_PR_Amount     += GLBudgetEntry."WEH_Encumbrance PR Amount";
            Out_PO_Amount     += GLBudgetEntry."WEH_Encumbrance PO Amount";
            Out_Actual_Amount += (GLBudgetEntry."WEH_Encumbrance PI Amount" +
                                  GLBudgetEntry."Encumbrance PCN Amt.");
        UNTIL GLBudgetEntry.NEXT() = 0;
    END;

    Out_Avaiable_Amount := Out_Budget_Amount - Out_PR_Amount
                           - Out_PO_Amount - Out_Actual_Amount;
END;
```

---

## 📎 อ้างอิงไฟล์ครบ

| ไฟล์ | บรรทัด | คำอธิบาย |
|------|-------|---------|
| `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` | 1011-1043 | `CheckBudget` — entry point ที่ปุ่มเรียก |
| `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` | 1045-1168 | `CheckBudgetByLine` — เรียกใช้ CheckAvailableBudget แล้วเขียน Encumbrance Ledger |
| `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` | 1170-1414 | **`CheckAvailableBudget`** — logic ตัดสิน Pass/Not Pass จริง ⭐ |
| `3 - Page/Pag50104.WEHAvailableBudgetFactbox.al` | 66-177 | **`GetBudgetInfo`** — logic FactBox ⭐ |
| `3 - Page/Pag50094.WEHPurchQuoteCorporate.al` | 1398-1432 | OnAction ปุ่ม Reserve Budget |
| `1 - Table/` (เกี่ยวข้อง) | – | `WEH_G/L Budget Enc. PR/PO` (50007) — Encumbrance Ledger ลึกสุด |
| `2 - Table Extend/Tab-Ext51009...` | – | `General Ledger Setup` extension (ฟิลด์ `WEH_G/L for Budget Type C`) |
| `2 - Table Extend/Tab-Ext51000...` | – | `G/L Budget Name` extension (ฟิลด์ `WEH_Encumbrance Name`, `Start/End Date`) |
