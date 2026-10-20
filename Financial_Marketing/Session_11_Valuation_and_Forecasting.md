# Session 11: Valuation & Forecasting – 1 (Free Cash Flow, FCFF & FCFE)

> 🎓 So far we have read statements (Sessions 1–2), judged them with ratios (3–4), studied costs and planning (5–8) and the cost of capital (9–10). Today we ask the biggest question in finance: **what is a business actually worth?** The answer starts with one idea: **cash, not profit**.

**By the end of this session you will be able to:**
- Explain free cash flow (FCF), FCFF, FCFE, DCF and valuation multiples
- Differentiate FCFF from FCFE, and P/E from Enterprise Value multiples
- Calculate FCFF and FCFE from basic financial data
- Apply DCF and multiples-based approaches to estimate company value
- Compare FCFF and FCFE valuations and see the impact of capital structure
- Analyse how discount rates, growth assumptions and multiples affect value

> ⚠️ The deck is "Valuation & Forecasting **1**". It covers valuation basics, FCF, FCFF and FCFE in detail. DCF and multiples are listed in the outcomes but not worked on the slides; Section 6 gives a short preview so the outcomes make sense.

---

## 1. What is Valuation?

**Valuation** = estimating the economic value of an asset, business or company.

$$\text{Value of an asset} = \text{Present value of the future economic benefits investors expect from it}$$

Valuation is based on:

| Driver | Question it answers |
|---|---|
| Future cash flows | How much cash will the business produce? |
| Earnings | How profitable is it? |
| Growth prospects | How fast can it grow? |
| Risk | How uncertain is the future? |
| Capital requirements | How much debt or equity does it need? |
| Market valuations of comparable companies | What are similar businesses worth today? |

---

## 2. Approaches and Uses of Valuation

| Approach | Core question | Typical tool |
|---|---|---|
| **Intrinsic valuation** | What are the company's future cash flows worth **today**? | DCF (Discounted Cash Flow) |
| **Relative valuation** | How is the market valuing **similar** companies? | Multiples (P/E, EV/EBITDA) |

**Uses of valuation:** investment decisions, Mergers & Acquisitions (M&A), IPO pricing, sale or purchase of a business, strategic decisions.

### Market Price ≠ Intrinsic Value

| | Market Price | Intrinsic Value |
|---|---|---|
| Meaning | The price at which the asset is **trading** | An **estimate** based on the asset's ability to generate future cash flows |
| Set by | Demand and supply, sentiment | Analysis of fundamentals |

> 💡 **In simple words:** Market price = dukaan pe likha hua daam. Intrinsic value = cheez asal mein kitne ki hai. Agar intrinsic value > market price, to share **undervalued** hai (sasta mil raha hai).

---

## 3. Factors that Affect the Value of a Business

| Factor | Question | Effect on value |
|---|---|---|
| **Cash flows** | How much cash can the business generate? | More cash → higher value |
| **Growth** | How quickly can cash flows grow? | Faster growth → higher value |
| **Risk** | How uncertain are future cash flows? | More risk → higher discount rate → lower value |
| **Investment requirements** | How much must the company reinvest? | More reinvestment → less free cash today |
| **Capital structure** | How much debt vs equity? | Changes the discount rate and the cash left for equity |
| **Terminal value** | What is the business worth beyond the forecast period? | Often the largest part of a DCF value |

---

## 4. Free Cash Flow (FCF)

Accounting profit measures **performance**; free cash flow measures the **cash available after necessary investment**.

**FCF** = the cash a business generates after making the investments needed to maintain and grow it.

$$\textbf{FCF = Operating Cash Flow − Capital Expenditure}$$

### Worked example: ABC Manufacturing Ltd (FY2026, ₹ crore)

| Particulars | ₹ crore |
|---|---|
| Net Profit after Tax | 1,000 |
| Depreciation | 200 |
| Increase in Inventory | 100 |
| Increase in Trade Receivables | 50 |
| Increase in Trade Payables | 40 |
| Capital Expenditure (machinery) | 300 |

At first glance the company "earned ₹1,000 crore". But because of accrual accounting, **cash ≠ profit**. We adjust step by step:

| Step | Adjustment | Why | Running total |
|---|---|---|---|
| Start | Net Profit | Starting point | 1,000 |
| 1 | **+** Depreciation 200 | Non-cash expense; no cash left the business | 1,200 |
| 2 | **−** Increase in Inventory 100 | Cash is tied up in stock | 1,100 |
| 3 | **−** Increase in Receivables 50 | Sales made, but cash not yet collected | 1,050 |
| 4 | **+** Increase in Payables 40 | Goods received but not paid for, so cash is retained | **1,090 = Operating Cash Flow** |
| 5 | **−** CapEx 300 | Actual cash paid for machinery | **790 = FCF** |

✔️ Check: FCF = 1,090 − 300 = **₹790 crore**. The company earned ₹1,000 crore of profit but only ₹790 crore of cash is truly "free".

**Working capital rule (remember this):**

| Change | Cash effect |
|---|---|
| Current **asset** ⬆️ (inventory, receivables) | Cash **outflow** (subtract) |
| Current **asset** ⬇️ | Cash **inflow** (add) |
| Current **liability** ⬆️ (payables) | Cash **inflow** (add) |
| Current **liability** ⬇️ | Cash **outflow** (subtract) |

> 💡 **In simple words:** Stock ya udhaar-pe-becha maal badha → paisa phasa (minus). Supplier ko payment abhi nahi kiya → paisa apne paas bacha (plus).

> ⚠️ Slide 11 shows "₹1,050 = ₹40 = ₹1,090"; it should be **₹1,050 + ₹40 = ₹1,090**. The same slide also repeats a stray line about receivables.
> ⚠️ Slide 13 says "Capital Expenditure = cash **inflow**". CapEx is a cash **outflow**, which is why it is subtracted.

---

## 5. FCFF and FCFE

### 5A. Free Cash Flow to Firm (FCFF)
Cash generated by operations that is available to **all providers of capital**: equity shareholders **and** debt holders. It is measured **before** any payment to either.

$$\textbf{FCFF} = \text{EBIT}(1 - T) + \text{Depreciation} - \text{CapEx} - \Delta\text{NWC}$$

| Term | Meaning |
|---|---|
| EBIT | Earnings Before Interest and Tax |
| T | Corporate tax rate |
| EBIT(1 − T) | Also called **NOPAT** (Net Operating Profit After Tax) |
| Depreciation | Non-cash expense, added back |
| CapEx | Capital expenditure |
| ΔNWC | Change in operating net working capital |

An alternative uses CFO (FCFF ≈ CFO − CapEx), but the **EBIT-based formula is preferred for valuation because it removes the effect of financing decisions** (interest never enters it).

### 5B. Free Cash Flow to Equity (FCFE)
Cash available **only to equity shareholders**, after meeting the cash-flow consequences of debt (interest and net borrowing).

$$\textbf{FCFE} = \text{NI} + \text{Depreciation} - \text{CapEx} - \Delta\text{NWC} + \text{Net Borrowing}$$
$$\text{Net Borrowing} = \text{New Debt Raised} - \text{Debt Repaid}$$

Alternative (shows the link with FCFF):
$$\textbf{FCFE} = \text{FCFF} - \text{Interest}(1 - T) + \text{Net Borrowing}$$

> 💡 **In simple words:** FCFF = poore pizza ka size (debt + equity dono ka). FCFE = lenders ko unka hissa (after-tax interest) dene aur naya loan/repayment adjust karne ke baad shareholders ke liye bacha hua pizza.

| | FCFF | FCFE |
|---|---|---|
| Belongs to | All capital providers (debt + equity) | Equity shareholders only |
| Interest | Not deducted | Deducted (after tax) |
| Net borrowing | Ignored | Added (new debt) / subtracted (repayment) |
| Starts from | EBIT(1 − T) | Net Income |
| Depends on capital structure? | No (largely) | Yes |
| No-debt company | FCFF = FCFE | FCFF = FCFE |

### 5C. Solved example (practice numbers)

Company X: EBIT ₹500 Cr, tax 25%, depreciation ₹80 Cr, CapEx ₹150 Cr, ΔNWC ₹30 Cr, interest ₹40 Cr, net borrowing +₹20 Cr.

| Step | Calculation | ₹ Cr |
|---|---|---|
| NOPAT | 500 × 0.75 | 375 |
| **FCFF** | 375 + 80 − 150 − 30 | **275** |
| After-tax interest | 40 × 0.75 | 30 |
| **FCFE (route 1)** | 275 − 30 + 20 | **265** |
| Net Income | (500 − 40) × 0.75 | 345 |
| **FCFE (route 2)** | 345 + 80 − 150 − 30 + 20 | **265** ✅ |

Both FCFE routes give ₹265 Cr, which confirms the link formula.

### 5D. Real-company examples from the slides

**Example 1: A low-debt, asset-light company (₹ crore)**

| | FY21 | FY22 | FY23 | FY24 | FY25 |
|---|---|---|---|---|---|
| Revenue | 1,64,177 | 1,91,754 | 2,25,458 | 2,40,893 | 2,55,324 |
| EBIT margin | 25.9% | 25.3% | 24.1% | 24.6% | 24.3% |
| Operating cash flow | 39,929 | 45,067 | 44,000 | 44,990 | 49,139 |
| CapEx | 3,119 | 2,876 | 3,155 | 3,063 | 2,650 |
| **FCFF ≈ CFO − CapEx** | 36,810 | 42,191 | 40,845 | 41,927 | 46,489 |
| Interest | ~600 | ~600 | ~700 | 778 | 796 |
| Net borrowing | ~0 | ~0 | ~0 | ~0 | ~0 |
| **FCFE ≈ FCFF − Interest + Net Borrowing** | 36,210 | 41,591 | 40,145 | 41,149 | 45,693 |

Lesson: CapEx is tiny compared to CFO, and there is almost no debt, so **FCFF and FCFE are nearly equal** and strongly positive. This is a classic cash machine.

> ⚠️ This table subtracts the **full** interest, while the formula slide uses Interest × (1 − T). With interest this small the difference is negligible, but in exams use the formula version unless told otherwise.

**Example 2: A capital-heavy conglomerate (₹ crore)**

| | FY21 | FY22 | FY23 | FY24 | FY25 |
|---|---|---|---|---|---|
| EBIT = EBITDA − D&A | 71,008 | 93,902 | 1,13,617 | 1,27,458 | 1,30,286 |
| NOPAT = EBIT × (1 − T) | 53,256 | 70,427 | 85,213 | 95,594 | 1,00,320 |
| + D&A | 26,572 | 29,782 | 40,303 | 50,832 | 53,136 |
| − CapEx | ~1,28,000 | ~1,09,000 | 1,41,809 | 1,31,769 | 1,39,967 |
| − ΔNWC | ~15,000 | ~10,000 | ~15,000 | ~6,366 | ~23,724 |
| **FCFF** | ~(63,172) | ~(18,791) | ~(31,293) | ~8,291 | ~(10,235) |
| CFO (reported) | — | — | 1,15,032 | 1,58,788 | 1,78,703 |
| Cash interest paid | — | — | ~21,650 | 37,173 | 41,024 |
| Net borrowing | — | — | ~38,075 | 9,262 | 19,247 |
| **FCFE = CFO − CapEx − Interest + Net Borrowing** | — | — | ~(10,352) | ~(892) | ~16,959 |

Lesson: profits are huge, but **CapEx is even bigger**, so FCFF is mostly **negative**. The company funds the gap with borrowing. Negative FCF is not automatically bad; it can mean heavy investment for future growth. The real question is whether those investments will earn more than their cost of capital.

> 💡 **In simple words:** Profit zyada hai, par company naye plant/projects mein usse bhi zyada paisa laga rahi hai. Isliye "free" cash negative hai aur gap loan se bhar rahi hai.

> ⚠️ The slide's FY25 FCFE cell is cut off ("17,?"). Using the slide's own figures: 1,78,703 − 1,39,967 − 41,024 + 19,247 = **₹16,959 Cr**. For FY24 the same figures give **~(892)**, not ~(833) as printed.

---

## 6. Preview: DCF and Multiples (from the learning outcomes)

**DCF (intrinsic):** value = present value of future cash flows.

$$PV = \frac{CF_t}{(1+r)^t} \qquad \text{Terminal Value}_n = \frac{FCF_{n+1}}{r - g}$$

| Cash flow discounted | Discount rate | Result |
|---|---|---|
| FCFF | WACC (Session 9) | **Enterprise Value** → Equity = EV − Debt + Cash |
| FCFE | Cost of equity | **Equity Value** directly |

**Multiples (relative):** P/E = Price ÷ EPS (equity multiple); EV/EBITDA (firm multiple, ignores capital structure). A higher discount rate or lower growth reduces DCF value; a higher peer multiple raises relative value.

---

## 7. Quick Revision 🧠

- Value = PV of expected future benefits. **Intrinsic** (DCF) vs **Relative** (multiples). Market price ≠ intrinsic value.
- Value drivers: cash flows, growth, risk, reinvestment, capital structure, terminal value.
- **FCF = OCF − CapEx.** OCF = NPAT + Dep − ↑CA + ↑CL.
- **FCFF = EBIT(1 − T) + Dep − CapEx − ΔNWC** → all capital providers.
- **FCFE = NI + Dep − CapEx − ΔNWC + Net Borrowing = FCFF − Interest(1 − T) + Net Borrowing** → equity only.
- No debt → FCFF = FCFE. Heavy CapEx → FCF can be negative even with big profits.
- FCFF @ WACC → EV; FCFE @ cost of equity → equity value.

---

## 8. MCQs (Mid–Advanced Level) 📝

**Q1.** A company reports NPAT ₹500 Cr, depreciation ₹60 Cr, increase in inventory ₹40 Cr, increase in receivables ₹30 Cr, increase in payables ₹20 Cr and CapEx ₹120 Cr. What is its free cash flow?
- A) ₹390 Cr
- B) ₹430 Cr
- C) ₹510 Cr
- D) ₹350 Cr

> ✅ **Answer: A.** OCF = 500 + 60 − 40 − 30 + 20 = 510. FCF = 510 − 120 = **₹390 Cr**. Option C is the operating cash flow before CapEx. 📖 Section 4

**Q2.** Trade payables increase by ₹40 Cr during the year. How is this treated when moving from net profit to operating cash flow?
- A) Subtracted, because it is a liability
- B) Subtracted, because it reduces profit
- C) Added, because the company has retained cash it has not yet paid
- D) Ignored, because it is a balance sheet item

> ✅ **Answer: C.** An increase in a current liability means goods/services were received without paying cash yet, so cash is retained. 📖 Section 4

**Q3.** Trade receivables **decrease** by ₹25 Cr during the year. What is the effect on operating cash flow?
- A) Increases by ₹25 Cr
- B) Decreases by ₹25 Cr
- C) Increases by ₹50 Cr
- D) No effect

> ✅ **Answer: A.** A fall in a current asset means customers paid off old dues, which brings cash in. 📖 Section 4

**Q4.** EBIT is ₹800 Cr, tax rate 30%, depreciation ₹100 Cr, CapEx ₹250 Cr and increase in NWC ₹50 Cr. What is FCFF?
- A) ₹460 Cr
- B) ₹410 Cr
- C) ₹360 Cr
- D) ₹600 Cr

> ✅ **Answer: C.** NOPAT = 800 × 0.7 = 560. FCFF = 560 + 100 − 250 − 50 = **₹360 Cr**. Option D ignores tax, Option A adds ΔNWC instead of subtracting it, and Option B forgets ΔNWC. 📖 Section 5A

**Q5.** FCFF is ₹360 Cr, interest expense ₹50 Cr, tax rate 30%, and the company **repaid** ₹40 Cr of debt (no new debt). What is FCFE?
- A) ₹285 Cr
- B) ₹325 Cr
- C) ₹270 Cr
- D) ₹365 Cr

> ✅ **Answer: A.** FCFE = 360 − 50 × 0.7 − 40 = 360 − 35 − 40 = **₹285 Cr**. Option C deducts pre-tax interest, Option B ignores the repayment, and Option D adds the repayment instead of subtracting it. 📖 Section 5B

**Q6.** Net income ₹300 Cr, depreciation ₹50 Cr, CapEx ₹120 Cr, increase in NWC ₹20 Cr, new debt raised ₹80 Cr and debt repaid ₹30 Cr. What is FCFE?
- A) ₹160 Cr
- B) ₹290 Cr
- C) ₹300 Cr
- D) ₹260 Cr

> ✅ **Answer: D.** Net borrowing = 80 − 30 = 50. FCFE = 300 + 50 − 120 − 20 + 50 = **₹260 Cr**. Option B uses gross new debt, Option C adds ΔNWC, and Option A subtracts net borrowing. 📖 Section 5B

**Q7.** Why is the EBIT-based FCFF formula preferred for valuation over a profit-after-interest measure?
- A) It always gives a higher number
- B) It includes interest, so it is more accurate
- C) It is required by accounting standards
- D) It removes the effect of financing decisions, so value does not depend on how the firm is funded

> ✅ **Answer: D.** Interest never enters EBIT(1 − T), so FCFF reflects operations only, not the debt/equity mix. 📖 Section 5A

**Q8.** For a company with **zero debt** (no interest, no borrowing or repayment), what is the relationship between FCFF and FCFE?
- A) FCFE cannot be calculated
- B) FCFF < FCFE
- C) FCFF = FCFE
- D) FCFF > FCFE

> ✅ **Answer: C.** With no interest and no net borrowing, FCFE = FCFF − 0 + 0. This is why the low-debt example in the slides shows almost equal values. 📖 Section 5D

**Q9.** A conglomerate has NOPAT of ₹1,00,000 Cr but negative FCFF because CapEx is ₹1,40,000 Cr. What is the best interpretation?
- A) The company is making accounting losses
- B) Negative FCFF means the company will go bankrupt
- C) The company is reinvesting heavily; whether this is good depends on the returns those projects earn versus their cost of capital
- D) The company is certainly overvalued

> ✅ **Answer: C.** Negative FCF from large growth CapEx is not automatically bad. Option A is wrong because NOPAT is large and positive. 📖 Section 5D

**Q10.** In the ABC Manufacturing example, NPAT was ₹1,000 Cr but FCF was ₹790 Cr. Which items caused the drop?
- A) CapEx and net working capital investment, partly offset by the depreciation add-back
- B) Only the increase in payables
- C) Only depreciation
- D) Interest and taxes

> ✅ **Answer: A.** +200 depreciation, −100 inventory, −50 receivables, +40 payables, −300 CapEx: net −210. 📖 Section 4

**Q11.** A share trades at ₹500. An analyst's DCF estimate of its intrinsic value is ₹620. What does this suggest?
- A) The share is overvalued
- B) The DCF is wrong because it differs from the price
- C) Market price must equal intrinsic value
- D) The share is undervalued relative to the analyst's estimate

> ✅ **Answer: D.** Intrinsic value above market price suggests undervaluation, though it is only an estimate based on assumptions. 📖 Section 2

**Q12.** Valuing a company by comparing its P/E with the P/E of similar listed companies is an example of:
- A) Relative valuation
- B) Intrinsic valuation
- C) Liquidation valuation
- D) Book value valuation

> ✅ **Answer: A.** Relative valuation asks how the market values similar companies, using multiples. 📖 Section 2

**Q13.** Why is depreciation added back to net profit when calculating operating cash flow?
- A) Because it is a cash inflow
- B) Because it is a non-cash expense that reduced profit without any cash leaving
- C) Because CapEx is already zero
- D) Because it increases taxes

> ✅ **Answer: B.** Cash left when the asset was bought (as CapEx); yearly depreciation is only an accounting allocation. 📖 Section 4

**Q14.** Net borrowing for the year: new debt raised ₹200 Cr and debt repaid ₹350 Cr. What is the effect on FCFE?
- A) No effect on FCFE
- B) FCFE decreases by ₹150 Cr
- C) FCFE increases by ₹200 Cr
- D) FCFE increases by ₹150 Cr

> ✅ **Answer: B.** Net borrowing = 200 − 350 = −150, so FCFE falls by ₹150 Cr. 📖 Section 5B

**Q15.** A cash flow of ₹1,210 will be received in 2 years. At a 10% discount rate, what is its present value?
- A) ₹1,089
- B) ₹1,000
- C) ₹968
- D) ₹1,100

> ✅ **Answer: B.** PV = 1,210 ÷ 1.1² = 1,210 ÷ 1.21 = **₹1,000**. 📖 Section 6

**Q16.** Next year's FCF after the forecast period is ₹110 Cr, the discount rate is 12% and the long-term growth rate is 2%. What is the terminal value?
- A) ₹1,100 Cr
- B) ₹785.7 Cr
- C) ₹5,500 Cr
- D) ₹916.7 Cr

> ✅ **Answer: A.** TV = 110 ÷ (0.12 − 0.02) = 110 ÷ 0.10 = **₹1,100 Cr**. Option D uses r alone (110 ÷ 0.12), and Option C uses g alone (110 ÷ 0.02). 📖 Section 6

**Q17.** If the discount rate used in a DCF increases (all else equal), what happens to the estimated value?
- A) It increases
- B) It doubles
- C) It decreases
- D) It stays the same

> ✅ **Answer: C.** A higher rate shrinks the present value of every future cash flow. Higher risk → higher discount rate → lower value. 📖 Sections 3 & 6

**Q18.** A DCF using FCFF and WACC gives an Enterprise Value of ₹5,000 Cr. Debt is ₹1,200 Cr and cash is ₹300 Cr. What is the equity value?
- A) ₹3,500 Cr
- B) ₹4,100 Cr
- C) ₹6,500 Cr
- D) ₹3,800 Cr

> ✅ **Answer: B.** Equity = EV − Debt + Cash = 5,000 − 1,200 + 300 = **₹4,100 Cr**. Option D ignores cash, and Option A subtracts cash. 📖 Section 6

**Q19.** Which pairing of cash flow and discount rate is correct?
- A) FCFE discounted at WACC gives enterprise value
- B) FCFF discounted at WACC gives enterprise value
- C) FCFE discounted at the cost of debt gives firm value
- D) FCFF discounted at cost of equity gives equity value

> ✅ **Answer: B.** Match the cash flow to its owners: FCFF (all capital providers) with WACC; FCFE (equity only) with the cost of equity. 📖 Section 6

**Q20.** A company has CFO ₹49,139 Cr, CapEx ₹2,650 Cr, interest ₹796 Cr and net borrowing of zero. Using the slide's approach (FCFE ≈ CFO − CapEx − Interest + Net Borrowing), what is FCFE?
- A) ₹46,489 Cr minus tax
- B) ₹51,789 Cr
- C) ₹46,489 Cr
- D) ₹45,693 Cr

> ✅ **Answer: D.** FCFF ≈ 49,139 − 2,650 = 46,489. FCFE ≈ 46,489 − 796 + 0 = **₹45,693 Cr**. Option C is FCFF. 📖 Section 5D

**Q21.** Which of the following is **not** one of the factors that affect the value of a business listed in this session?
- A) Terminal value
- B) Face value of the company's shares
- C) Risk of future cash flows
- D) Reinvestment requirements

> ✅ **Answer: B.** Face value is a legal/accounting figure on the share certificate and does not drive economic value. 📖 Section 3

**Q22.** Two companies have identical operations and identical FCFF, but Company P has no debt and Company Q is heavily financed with debt. Which statement is most accurate?
- A) Capital structure affects FCFF but not FCFE
- B) Their FCFE will be identical
- C) Q will always have higher FCFF
- D) Their FCFF is the same, but their FCFE will differ because Q pays interest and has borrowing flows

> ✅ **Answer: D.** FCFF is independent of financing; FCFE depends on interest and net borrowing, so capital structure changes it. Option A has it backwards. 📖 Section 5B
