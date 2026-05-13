# เปรียบเทียบ logic: ปุ่ม Reserve Budget vs. FactBox Available Budget

> เอกสารสำหรับทีม BA
> ขอบเขต: หน้า WEH_Purch. Quote - Corporate (Page 50094)
> วันที่: 2026-05-13

---

## 🎯 สรุปสั้น

- **ปุ่ม Reserve Budget** เช็คผ่าน/ไม่ผ่าน → ทำที่ `Cod50009.CheckAvailableBudget`
- **FactBox โชว์ Available** → ทำที่ `Pag50104.GetBudgetInfo`
- ทั้งสองอ่าน Table เดียวกัน **แต่ใช้ filter ต่างกัน** → ตัวเลขอาจไม่ตรงกัน
- **FactBox = ดูประมาณ** ไม่ใช่ตัวตัดสิน
- **CheckAvailableBudget = ตัวจริงที่ตัดสิน Pass / Not Pass** เข้มกว่า ละเอียดกว่า

---

## 📊 Table ที่ใช้ทั้ง 2 ฝั่ง

| Table | ใช้ทำอะไร |
|-------|----------|
| `Purchase Line` | บรรทัดเอกสาร |
| `G/L Budget Entry` (Budget Name = WEH_Budget Name) | งบประมาณตั้งต้น |
| `G/L Budget Entry` (Budget Name = WEH_Encumbrance Name) | ตัวจองงบ (PR / PO / PI / PCN) |
| `G/L Entry` | รายการโพสจริง |
| `General Ledger Setup` | ตั้งค่า G/L กลางสำหรับ Type C |
| `G/L Budget Name` | ผูก Budget กับ Encumbrance Name |

---

## ✅ การเช็ค Pass — `CheckAvailableBudget` (Cod50009 บรรทัด 1170-1414)

**สูตร**: `Available = Budget − Encumbrance − Actual`
**ผ่านเมื่อ**: ผลรวมของ Line ในเอกสารที่ match ≤ Available

### Type O (งบโครงการ / Operational)

| รายการ | Filter |
|--------|--------|
| **Budget** | `G/L Budget Entry`: Budget Name = Line.`WEH_Budget Name` + G/L Acc = `TEMP_ACC_NO*` + Dim1 + `WEH_Budget Type = O` |
| **Encumbrance** | `G/L Budget Entry` (Encumbrance Name) + G/L Acc = `TEMP_ACC_NO` + Dim1 + Type = O<br>→ Σ (`WEH_Encumbrance PR Amount` + `PO Amount` + `PI Amount`) |
| **Actual** | `G/L Entry`: G/L Acc = `RBG_Ref. G/L (Item/FA)-Budget` + Dim1 + Posting Date ∈ [Start, End] |

`*TEMP_ACC_NO`:
- ถ้า `Line.Type = "G/L Account"` → `Line."No."`
- อื่น ๆ (Item / FA / Resource) → `Line."RBG_Ref. G/L (Item/FA)-Budget"`

### Type C (งบกลาง / Central)

| รายการ | Filter |
|--------|--------|
| **Budget** | `G/L Budget Entry`: Budget Name + G/L Acc = `GLSetup."WEH_G/L for Budget Type C"` + **Dim2** |
| **Encumbrance** | `G/L Budget Entry` (Encumbrance Name) + **Dim1 + Dim2** + Type = C<br>→ Σ (PR + PO + PI) |
| **Actual** | `G/L Entry` + Dim1 + Dim2 + Date Range<br>**ข้าม** G/L ที่ `WHE_SkipCheckBudget = true` |

### Type P
→ **Pass เลย** ไม่เช็ค

### Logic เปรียบเทียบ
```
PRInLineAmt = Σ Cal_AmountPurchaseLine(Line ทุกบรรทัดในเอกสาร
                                       ที่ Dim + Budget Type + Budget Name ตรงกัน)

IF PRInLineAmt <= AvaiableBudget THEN
    Line.WEH_Check Budget := Pass
ELSE
    Line.WEH_Check Budget := Not Pass
```

---

## 🪟 FactBox โชว์ — `GetBudgetInfo` (Pag50104)

โชว์ของ **บรรทัดเดียวที่เลือกอยู่** (Rec = Purchase Line ปัจจุบัน)
**สูตร**: `Available = Budget − PR − PO − Actual`

### Type O

| ช่อง | Filter |
|------|--------|
| **Budget** | Budget Name + G/L Acc = `Rec."RBG_Ref. G/L"` + Dim1 + Type = O |
| **Reserve Budget (PR)** | Encumbrance Name + G/L Acc = `RBG_Ref. G/L` + Dim1<br>→ Σ `WEH_Encumbrance PR Amount` |
| **Obligate Budget (PO)** | เหมือน PR → Σ `WEH_Encumbrance PO Amount` |
| **Actual** | เหมือน PR → Σ (`WEH_Encumbrance PI Amount` + `Encumbrance PCN Amt.`) |

### Type C

| ช่อง | Filter |
|------|--------|
| **Budget** | Budget Name + **Dim2 อย่างเดียว** (ไม่ filter G/L Account) |
| **PR / PO / Actual** | Encumbrance Name + **Dim2 อย่างเดียว** (ไม่ filter G/L Account, ไม่ filter Dim1) |

---

## ⚠️ 7 จุดที่ทำให้ตัวเลข **ไม่ตรงกัน**

| # | เรื่อง | ตัวเช็ค Pass (`CheckAvailableBudget`) | FactBox (`GetBudgetInfo`) | ผลที่ตามมา |
|---|------|--------------------------------------|--------------------------|----------|
| 1 | **Actual มาจากไหน** | `G/L Entry` (โพสจริง) | `PI + PCN` จาก Encumbrance | โพสจริง ≠ PI Encumbrance → ตัวเลขไม่ตรง |
| 2 | **Type O: G/L Account** | ขึ้นกับ `Line.Type` (G/L → ใช้ `No.`, อื่น ๆ → `RBG_Ref. G/L`) | ใช้ `RBG_Ref. G/L` เสมอ | กรณี Line เป็น G/L Account → filter ต่าง |
| 3 | **Type C: Budget filter** | filter G/L จาก `GLSetup` | ไม่ filter G/L | FactBox อาจโชว์ Budget มากเกินจริง |
| 4 | **Type C: Encumbrance filter** | Dim1 + Dim2 | Dim2 อย่างเดียว | FactBox รวมทุก Dim1 |
| 5 | **Type C: Skip บัญชี** | ข้าม G/L ที่ `WHE_SkipCheckBudget = true` | ไม่มี logic นี้ | FactBox นับครบทุกบัญชี |
| 6 | **ระดับการรวม** | รวมทุก Line ของเอกสารแล้วเทียบ | โชว์ pool เฉย ๆ ไม่บอกว่าใช้ไปเท่าไหร่ | FactBox ตอบไม่ได้ว่าจะผ่านหรือไม่ |
| 7 | **PCN** | ไม่นับ | นับใน Actual | ค่าต่างกันถ้ามี PCN |

---

## 💡 ข้อเสนอแนะสำหรับ BA

1. **อย่าใช้ตัวเลขใน FactBox ตัดสินใจว่าจะผ่านหรือไม่** — ให้กดปุ่ม Reserve Budget แล้วดูผลจริง
2. ถ้าผู้ใช้ถามว่า "ทำไม FactBox โชว์ว่ามีเงิน แต่ Reserve ไม่ผ่าน" → คำตอบอยู่ใน 7 ข้อด้านบน (มักเป็นข้อ 1, 4, 5)
3. ถ้าต้องการให้ FactBox ตรงกับ logic เช็คจริง → ต้องแก้ `Pag50104.GetBudgetInfo` ให้ใช้ filter เดียวกับ `CheckAvailableBudget` (เป็น scope งานแยก)

---

## 📎 อ้างอิงไฟล์

| ไฟล์ | บรรทัด | คำอธิบาย |
|------|-------|---------|
| `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` | 1011-1043 | `CheckBudget` — entry point |
| `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` | 1045-1168 | `CheckBudgetByLine` |
| `5 - Codeunit/Cod50009.WEHBudgetControlManagement.al` | 1170-1414 | `CheckAvailableBudget` — logic เช็คจริง |
| `3 - Page/Pag50104.WEHAvailableBudgetFactbox.al` | 66-177 | `GetBudgetInfo` — logic FactBox |
| `3 - Page/Pag50094.WEHPurchQuoteCorporate.al` | 1398-1432 | OnAction ของปุ่ม Reserve Budget |
