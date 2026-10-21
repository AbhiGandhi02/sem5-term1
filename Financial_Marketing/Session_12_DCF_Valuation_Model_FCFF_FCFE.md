# Session 12: DCF Valuation Model – FCFF (Reliance) & FCFE (HDFC Bank)

> 🎓 In Session 11 we learnt what FCFF and FCFE are. Session 9 gave us CAPM, and Session 7 gave us discounting and terminal value. Today we **put everything together in one spreadsheet model** and turn cash flows into a **share price**.

**By the end of this session you will be able to:**
- Read a DCF workbook: inputs vs formulas vs links (colour legend)
- Build base-year FCFF from EBIT, and base-year FCFE from Net Income (bank version)
- Calculate Cost of Equity (CAPM) and WACC
- Project 5 years of cash flows, discount them and add a terminal value
- Bridge Enterprise Value → Equity Value → Value per Share
- Explain why Reliance is valued with FCFF/WACC and HDFC Bank with FCFE/Ke

> ⚠️ The source is a **4-page Excel model printout ("SHEET")**, not slides. All figures are **illustrative round numbers**, not real Reliance/HDFC data. The sheet prints ₹ Crore values with a **"$" sign**; read them all as ₹ Crore.

> ⚠️ **Big error in the sheet:** in both models the terminal value is discounted with the **Year 4** factor instead of Year 5. The corrected values are worked out in Sections 4 and 5. Use the corrected method in exams.

---

## 1. How the Workbook is Organised

| Sheet | What it does |
|---|---|
| **Read Me** | Legend, structure, disclaimer |
| **Reliance FCFF** | EBIT → NOPAT → FCFF → WACC → Enterprise Value → Equity Value → Share Price |
| **HDFC Bank FCFE** | Net Income → FCFE → Cost of Equity (CAPM) → Equity Value directly → Share Price |
| **Comparison** | Side-by-side summary of both models, linked live |

### Colour legend (standard financial-modelling convention)

| Colour | Meaning | Rule |
|---|---|---|
| 🟨 **Yellow fill** | Hard-coded **input** (growth rates, WACC drivers, capex…) | The only cells you should change |
| ⬛ **Black text** | **Formula**, calculated automatically | Never overwrite; it breaks the chain |
| 🟩 **Green text** | **Link** pulling a value from another sheet | Used on the Comparison tab |

> 💡 **In simple words:** Yellow = tumhara assumption, sirf yahi badlo. Black = Excel khud calculate karta hai, haath mat lagao. Green = doosri sheet se aaya hua number.

**The DCF chain (both models):**
Base-year cash flow → grow it for 5 years → discount each year → add discounted terminal value → (FCFF only: subtract net debt) → divide by shares.

---

## 2. Model A: Reliance Industries – FCFF / WACC (₹ Crore)

### 2A. Base-year (Year 0) FCFF build

$$\text{FCFF} = \text{EBIT}(1-T) + \text{D\&A} - \text{Capex} - \Delta\text{NWC}$$

| Line | Amount |
|---|---|
| EBIT (Year 0) | 90,000 |
| Tax rate | 25% |
| **NOPAT** = 90,000 × (1 − 0.25) | **67,500** |
| Add: Depreciation & Amortisation | + 35,000 |
| Less: Capex | − 55,000 |
| Less: Increase in NWC | − 5,000 |
| **FCFF (Year 0)** | **42,500** ✅ |

- **NOPAT** (Net Operating Profit After Tax) = operating profit as if the company had **no debt**. Interest is ignored because FCFF belongs to **both** lenders and shareholders.
- D&A is added back because it's non-cash. Capex and the NWC increase are real cash going out.

### 2B. WACC

| Input | Value |
|---|---|
| Risk-free rate (Rf) | 7.0% |
| Beta | 1.10 |
| Equity Risk Premium (ERP) | 6.0% |
| **Cost of Equity** Re = 7% + 1.10 × 6% | **13.6%** |
| Pre-tax cost of debt (Rd) | 8.0% |
| **After-tax Rd** = 8% × (1 − 0.25) | **6.0%** |
| Weight of equity (E/V) | 75% |
| Weight of debt (D/V) | 25% (check: weights sum to 100% → "OK") |
| **WACC** = 0.75 × 13.6% + 0.25 × 6.0% = 10.2% + 1.5% | **11.7%** ✅ |

$$\text{WACC} = W_e \times R_e + W_d \times R_d(1-T)$$

> 💡 **In simple words:** WACC = company ke saare paise ka average "kiraya". 75% paisa shareholders ka (13.6% maangte hain), 25% lenders ka (tax bachat ke baad sirf 6%). Weighted average = 11.7%.

### 2C. 5-year FCFF projection (WACC = 11.7%)

| | Year 0 | Year 1 | Year 2 | Year 3 | Year 4 | Year 5 |
|---|---|---|---|---|---|---|
| Growth (y/y) | | 10% | 9% | 8% | 7% | 6% |
| FCFF | 42,500 | 46,750 | 50,958 | 55,034 | 58,886 | 62,420 |
| Discount factor 1/(1.117)ⁿ | 1.000 | 0.895 | 0.801 | 0.718 | 0.642 | 0.575 |
| PV of FCFF | | 41,853 | 40,842 | 39,489 | 37,827 | 35,897 |

**Sum of PV (Years 1–5) = ₹1,95,907 Cr** ✅

Notice that growth **fades** from 10% to 6%, so it can slide smoothly into the 5% terminal growth.

---

## 3. Terminal Value (TV)

A company doesn't stop after Year 5, so we capture everything after that in one number using the **Gordon growth formula**:

$$\text{TV}_5 = \frac{\text{FCFF}_5 \times (1+g)}{\text{WACC} - g}$$

Reliance: TV₅ = 62,420 × 1.05 ÷ (0.117 − 0.05) = **₹9,78,219 Cr** ✅

**Key rules:**
- TV is the value **at the end of Year 5**, so it must be discounted by **(1 + WACC)⁵**.
- g must be **less than** WACC; otherwise the formula breaks (division by zero or a negative number).
- Long-run g should not exceed long-run nominal economic growth; no firm can outgrow the economy forever.

> 💡 **In simple words:** Year 5 ke baad ki saari "forever" kamai ko ek bada number bana diya: TV. Ye number Year 5 ke end pe khada hai, isliye ise 5 saal peeche discount karna hai, 4 nahi.

---

## 4. Reliance: Enterprise Value → Equity Value → Share Price

| Step | Sheet value | **Corrected** value |
|---|---|---|
| Sum of PV of FCFF (Years 1–5) | 1,95,907 | 1,95,907 |
| PV of Terminal Value | 6,28,381 ❌ (÷ 1.117⁴) | **5,62,562** (÷ 1.117⁵ = × 0.575) |
| **Enterprise Value** | 8,24,289 | **7,58,469** |
| Less: Net Debt | − 1,00,000 | − 1,00,000 |
| **Equity Value** | 7,24,289 | **6,58,469** |
| Shares outstanding (Crore) | 676 | 676 |
| **Implied value per share** | ₹1,071.43 | **₹974.07** |

$$\text{Equity Value} = \text{EV} - \text{Net Debt}, \qquad \text{Value per share} = \frac{\text{Equity Value}}{\text{Shares}}$$

> ⚠️ The sheet's "PV of Terminal Value" 6,28,381 = 9,78,219 ÷ 1.117⁴, i.e. the **Year 4** factor (0.642). The correct factor is Year 5 (0.575). The error overstates the share price by about 10%.

- ₹ Crore ÷ Crore shares = ₹ per share (the units cancel).
- **Net Debt** = Debt − Cash. FCFF values the **whole firm** (EV), so the lenders' claim is removed to get the shareholders' part.
- In the corrected model, TV is **~74% of EV**. Most of a DCF value comes from the terminal value, which is why g and WACC assumptions matter so much.

**Sensitivity (corrected model):**

| Change | Value per share |
|---|---|
| Base case (WACC 11.7%, g 5%) | ₹974.07 |
| Terminal g 6% | ₹1,129.38 |
| WACC 12.7% | ₹827.00 |
| WACC 10.7% | ₹1,172.80 |

A 1-point change in WACC or g moves the price by roughly 15–20%.

---

## 5. Model B: HDFC Bank – FCFE / Cost of Equity (₹ Crore)

### 5A. Base-year (Year 0) FCFE build (bank version)

| Line | Amount |
|---|---|
| Net Income | 60,000 |
| Add: D&A | + 2,000 |
| Less: Capex | − 3,000 |
| Less: Net investment in loan book / working capital | − 8,000 |
| Add: Net borrowing (change in subordinated debt) | + 1,000 |
| **FCFE (Year 0)** | **52,000** ✅ |

- For a bank, **"working capital" is replaced by loan-book growth**. Every new loan uses up cash.
- Regulatory capital requirements (Basel/RBI) often reduce distributable FCFE further.

### 5B. Cost of equity (the discount rate)

$$R_e = R_f + \beta \times \text{ERP} = 7\% + 0.90 \times 6\% = \mathbf{12.4\%}$$

**No WACC step for banks:** a bank's "debt" (deposits and borrowings) is the **raw material of its business**, not a separate financing layer. So there's no meaningful Enterprise Value or WACC. We value equity directly.

> 💡 **In simple words:** Reliance ke liye loan sirf funding hai, isliye pehle poori company (EV) value karo phir debt ghatao. Bank ke liye deposits hi uska "maal" hai, jaise kirane wale ka stock. Isliye seedha shareholders ka cash (FCFE) lo aur Ke se discount karo.

### 5C. 5-year FCFE projection (Ke = 12.4%)

| | Year 0 | Year 1 | Year 2 | Year 3 | Year 4 | Year 5 |
|---|---|---|---|---|---|---|
| Growth (y/y) | | 14% | 13% | 12% | 11% | 10% |
| FCFE | 52,000 | 59,280 | 66,986 | 75,025 | 83,277 | 91,605 |
| Discount factor 1/(1.124)ⁿ | 1.000 | 0.890 | 0.792 | 0.704 | 0.627 | 0.557 |
| PV of FCFE | | 52,740 | 53,022 | 52,833 | 52,175 | 51,061 |

**Sum of PV (Years 1–5) = ₹2,61,831 Cr** ✅

### 5D. Terminal value and equity value

TV₅ = 91,605 × 1.06 ÷ (0.124 − 0.06) = **₹15,17,212 Cr** ✅

| Step | Sheet value | **Corrected** value |
|---|---|---|
| Sum of PV of FCFE | 2,61,831 | 2,61,831 |
| PV of Terminal Value | 9,50,563 ❌ (÷ 1.124⁴) | **8,45,697** (÷ 1.124⁵) |
| **Equity Value** (no net-debt step) | 12,12,394 | **11,07,528** |
| Shares (Crore) | 555 | 555 |
| **Implied value per share** | ₹2,184.49 | **₹1,995.55** |

> ⚠️ Same mistake as Reliance: the TV is discounted with the Year 4 factor (0.627) instead of Year 5 (0.557). Also, the section heading says "Equity Value Per **Price**"; it means "Per **Share**".

---

## 6. FCFF vs FCFE – Side-by-Side

| | Reliance (FCFF) | HDFC Bank (FCFE) |
|---|---|---|
| Cash flow | Free cash flow to **firm** (lenders + shareholders) | Free cash flow to **equity** only |
| Discount rate | **WACC** 11.7% | **Cost of equity** 12.4% |
| Base-year cash flow | ₹42,500 Cr | ₹52,000 Cr |
| Terminal growth | 5% | 6% |
| Sum PV of explicit flows | ₹1,95,907 Cr | ₹2,61,831 Cr |
| Gives you | **Enterprise Value** → minus net debt → equity | **Equity Value** directly |
| Value per share (sheet → corrected) | ₹1,071 → **₹974** | ₹2,184 → **₹1,996** |

**Why the methods differ:**
- **Reliance:** leverage has swung a lot (Jio debt build-up, then large equity stake sales). FCFF/WACC values the whole enterprise first and handles debt separately as net debt.
- **HDFC Bank:** deposits and borrowings are the core business, so there's no clean "pre-interest" operating line. FCFE is discounted straight to equity.
- **Consistency check:** with fully consistent assumptions, FCFF-based and FCFE-based equity values for the **same** company should **converge**. A big gap signals inconsistent inputs.

**Matching rule (never break it):** FCFF ↔ WACC, FCFE ↔ Cost of equity. Discounting FCFE at WACC (a lower rate) would **overstate** equity value.

---

## 7. Quick Revision 🧠

- **NOPAT** = EBIT × (1 − T)
- **FCFF** = NOPAT + D&A − Capex − ΔNWC
- **FCFE (bank)** = NI + D&A − Capex − Net loan-book investment + Net borrowing
- **Re (CAPM)** = Rf + β × ERP; **WACC** = We·Re + Wd·Rd(1 − T)
- **Discount factor** = 1 ÷ (1 + r)ⁿ
- **TV₅** = CF₅ × (1 + g) ÷ (r − g), discounted by **(1 + r)⁵**
- **FCFF route:** EV = ΣPV + PV(TV) → Equity = EV − Net debt → ÷ shares
- **FCFE route:** Equity = ΣPV + PV(TV) → ÷ shares (banks: no EV/WACC)
- TV is usually **70%+ of value**, so g and the discount rate drive the answer
- Yellow = input, black = formula, green = link

---

## 8. MCQs (Mid–Advanced Level) 📝

**Q1.** A company has EBIT of ₹500 Cr, a 25% tax rate, D&A of ₹80 Cr, capex of ₹150 Cr and an increase in net working capital of ₹30 Cr. What is its FCFF?
- A) ₹335 Cr
- B) ₹275 Cr
- C) ₹400 Cr
- D) ₹195 Cr

> ✅ **Answer: B.** NOPAT = 500 × 0.75 = 375. FCFF = 375 + 80 − 150 − 30 = **₹275 Cr**. Option A adds the NWC increase instead of subtracting it, Option C uses EBIT instead of NOPAT, and Option D forgets to add back D&A. 📖 Section 2A

**Q2.** Why is NOPAT, rather than Net Income, the starting point for FCFF?
- A) Tax is not relevant for FCFF
- B) Net Income includes depreciation, which must be excluded
- C) FCFF is the cash available to both lenders and shareholders, so it is measured before interest
- D) NOPAT is always larger than Net Income

> ✅ **Answer: C.** FCFF belongs to all capital providers, so it is calculated as if the company had no debt: EBIT × (1 − T). Interest is reflected later through the after-tax cost of debt in WACC. 📖 Section 2A

**Q3.** Cost of equity is 12%, pre-tax cost of debt is 10%, the tax rate is 25%, and the capital structure is 60% equity and 40% debt. What is the WACC?
- A) 10.2%
- B) 12.0%
- C) 11.2%
- D) 9.3%

> ✅ **Answer: A.** After-tax Kd = 10% × 0.75 = 7.5%. WACC = 0.6 × 12% + 0.4 × 7.5% = 7.2% + 3.0% = **10.2%**. Option C uses the pre-tax cost of debt, and Option D swaps the weights. 📖 Section 2B

**Q4.** Rf = 6.5%, beta = 0.8 and the equity risk premium = 7%. What is the cost of equity?
- A) 12.1%
- B) 12.4%
- C) 11.7%
- D) 13.5%

> ✅ **Answer: A.** Re = 6.5% + 0.8 × 7% = 6.5% + 5.6% = **12.1%**. Option D adds the full ERP without multiplying by beta. 📖 Section 5B

**Q5.** Year-5 free cash flow is ₹1,000 Cr, terminal growth is 4% and WACC is 12%. What is the terminal value at the end of Year 5?
- A) ₹13,000 Cr
- B) ₹6,250 Cr
- C) ₹12,500 Cr
- D) ₹8,666.67 Cr

> ✅ **Answer: A.** TV = 1,000 × 1.04 ÷ (0.12 − 0.04) = 1,040 ÷ 0.08 = **₹13,000 Cr**. Option C forgets to grow the cash flow by (1 + g), and Option D divides by WACC instead of (WACC − g). 📖 Section 3

**Q6.** A terminal value of ₹20,000 Cr is calculated at the end of Year 5, and WACC is 10%. What is its present value?
- A) ₹13,660.27 Cr
- B) ₹12,418.43 Cr
- C) ₹20,000 Cr
- D) ₹18,181.82 Cr

> ✅ **Answer: B.** PV = 20,000 ÷ 1.1⁵ = **₹12,418.43 Cr**. Option A discounts for only 4 years, which is exactly the mistake in the class sheet. Option D discounts for only 1 year. 📖 Section 3

**Q7.** In the Reliance sheet, the terminal value of ₹9,78,219 Cr is shown with a PV of ₹6,28,381 Cr. What is the correct PV at a WACC of 11.7%?
- A) ₹6,28,381 Cr
- B) ₹8,75,756 Cr
- C) ₹9,78,219 Cr
- D) ₹5,62,562 Cr

> ✅ **Answer: D.** TV sits at the end of Year 5, so PV = 9,78,219 × 0.575 (= 1/1.117⁵) = **₹5,62,562 Cr**. Option A is the sheet's figure, which uses the Year 4 factor (0.642). 📖 Section 4

**Q8.** Enterprise value is ₹8,000 Cr, net debt is ₹1,500 Cr and there are 50 Cr shares. What is the implied value per share?
- A) ₹30
- B) ₹130
- C) ₹160
- D) ₹190

> ✅ **Answer: B.** Equity = 8,000 − 1,500 = 6,500. Per share = 6,500 ÷ 50 = **₹130**. Option C ignores net debt, and Option D adds net debt instead of subtracting it. 📖 Section 4

**Q9.** Why is HDFC Bank valued with FCFE and the cost of equity instead of FCFF and WACC?
- A) For a bank, deposits and borrowings are its raw material, not a separate financing layer, so EV and WACC are not meaningful
- B) Banks have no shareholders' equity
- C) Banks do not pay tax
- D) FCFE always gives a higher value than FCFF

> ✅ **Answer: A.** A bank's "debt" is part of its operations, so there is no clean pre-interest operating cash flow. The model values equity directly. 📖 Section 5B

**Q10.** A bank has Net Income ₹1,000 Cr, D&A ₹50 Cr, capex ₹80 Cr, net investment in its loan book ₹300 Cr and net borrowing (subordinated debt) of ₹40 Cr. What is its FCFE?
- A) ₹1,310 Cr
- B) ₹630 Cr
- C) ₹1,010 Cr
- D) ₹710 Cr

> ✅ **Answer: D.** FCFE = 1,000 + 50 − 80 − 300 + 40 = **₹710 Cr**. Option B subtracts net borrowing instead of adding it, and Option C ignores the loan-book investment. 📖 Section 5A

**Q11.** For a bank, which item takes the place of "increase in net working capital" in the FCFE build?
- A) Dividends paid
- B) Interest paid on deposits
- C) Net investment in the loan book
- D) Depreciation

> ✅ **Answer: C.** Growing the loan book ties up cash, just as rising inventory and receivables do for a manufacturer. 📖 Section 5A

**Q12.** In the Reliance model, what does the WACC check "weights sum to 100% → OK" protect against?
- A) A negative cost of debt
- B) Using the wrong number of shares
- C) A capital-structure input error, where equity and debt weights don't add up to the whole firm
- D) A terminal growth rate above WACC

> ✅ **Answer: C.** WACC is a weighted average, so the weights must sum to 100%. Built-in checks catch input mistakes before they flow through the model. 📖 Section 2B

**Q13.** What is the discount factor for Year 3 at a rate of 12%?
- A) 0.8929
- B) 0.7118
- C) 0.7972
- D) 0.6400

> ✅ **Answer: B.** 1 ÷ 1.12³ = 1 ÷ 1.4049 = **0.7118**. Option A is the Year 1 factor, and Option C is the Year 2 factor. 📖 Section 2C

**Q14.** In the corrected Reliance model, PV of terminal value is about 74% of enterprise value. What is the main implication?
- A) The company is overvalued
- B) The valuation is highly sensitive to the terminal growth rate and WACC
- C) Net debt should be ignored
- D) The 5-year forecast is irrelevant and can be skipped

> ✅ **Answer: B.** When most of the value sits in the terminal value, small changes in g or WACC swing the answer a lot. In the corrected model, raising g from 5% to 6% lifts the value from ₹974 to about ₹1,129 per share. 📖 Section 4

**Q15.** What happens to the implied share price if WACC rises from 11.7% to 12.7% while everything else stays the same?
- A) It does not change, because only the terminal growth rate matters
- B) It rises, because a higher WACC means a riskier, higher-return company
- C) It falls only through the 5-year forecast, not through the terminal value
- D) It falls, because every future cash flow is discounted more heavily and the terminal value shrinks

> ✅ **Answer: D.** A higher discount rate lowers the PV of each year and also raises the (WACC − g) denominator, so the TV falls too. In the corrected Reliance model the value drops from ₹974 to about ₹827. Option C is wrong because the TV is hit hardest. 📖 Section 4

**Q16.** Why must the terminal growth rate be lower than the discount rate?
- A) If g ≥ r, the Gordon formula's denominator is zero or negative, giving an infinite or meaningless value
- B) Because the discount rate already includes inflation
- C) Because the terminal value must always be smaller than the 5-year PV sum
- D) Because regulators cap growth rates

> ✅ **Answer: A.** TV = CF × (1 + g) ÷ (r − g) only works when r > g. Economically, no company can grow faster than its cost of capital forever. 📖 Section 3

**Q17.** An analyst discounts FCFE at the company's WACC instead of its cost of equity. What is the likely effect?
- A) Equity value is understated
- B) Equity value is overstated, because WACC is usually lower than the cost of equity
- C) There is no effect, because both rates are the same
- D) Enterprise value is calculated correctly

> ✅ **Answer: B.** WACC blends in cheaper after-tax debt, so it is below Ke. Discounting equity cash flows at a lower rate inflates their value. Always match FCFE with Ke and FCFF with WACC. 📖 Section 6

**Q18.** In a correctly built FCFF model, how do you get from enterprise value to equity value?
- A) Multiply by the cost of equity
- B) Add net debt
- C) Subtract the terminal value
- D) Subtract net debt (debt − cash)

> ✅ **Answer: D.** EV is the value of the whole firm, which belongs to lenders and shareholders. Removing the lenders' net claim leaves the shareholders' equity. 📖 Section 4

**Q19.** In the workbook's colour legend, which cells should a user change when running scenarios?
- A) Yellow-filled input cells
- B) Any cell, as long as the total stays the same
- C) Green-text link cells
- D) Black-text formula cells

> ✅ **Answer: A.** Yellow cells are hard-coded assumptions. Overwriting black formula cells breaks the calculation chain, and green cells are links from other sheets. 📖 Section 1

**Q20.** If FCFF and FCFE models for the same company use fully consistent assumptions, what should happen to the two equity values?
- A) FCFE should always be exactly double FCFF
- B) They cannot be compared
- C) They should converge to roughly the same number
- D) FCFF should always be higher, because it includes debt

> ✅ **Answer: C.** Both methods value the same shareholders' claim. A large gap means the assumptions (growth, leverage, discount rates) are inconsistent between the two models. 📖 Section 6

**Q21.** Reliance's Year 0 FCFF is ₹42,500 Cr and Year 1 growth is 10%. What is the PV of Year 1 FCFF at a WACC of 11.7%?
- A) ₹42,500 Cr
- B) ₹46,750 Cr
- C) ₹41,853 Cr
- D) ₹38,048 Cr

> ✅ **Answer: C.** FCFF₁ = 42,500 × 1.10 = 46,750. PV = 46,750 ÷ 1.117 = **₹41,853 Cr**. Option B is the undiscounted cash flow, and Option D discounts the Year 0 figure instead of the Year 1 figure. 📖 Section 2C

**Q22.** Why does the Reliance model's growth rate fade from 10% in Year 1 to 6% in Year 5 before a 5% terminal rate?
- A) Because tax rates rise each year
- B) Because the discount factor requires falling growth
- C) To make the terminal value larger
- D) So that the explicit forecast slides smoothly into a stable, long-run growth rate that is sustainable forever

> ✅ **Answer: D.** High growth can't last forever. Fading it toward the terminal rate avoids a sudden jump between Year 5 and the perpetuity assumption. 📖 Section 2C
