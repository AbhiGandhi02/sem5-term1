# Session 7: Financial Planning & Analysis – 3 (Time Value of Money)

> 🎓 Session 5 covered costs and decisions. Today we learn the most important tool in all of finance: **₹1 today is worth more than ₹1 tomorrow**. Every valuation, loan, bond and project decision you will ever see is built on this idea.

**By the end of this session you will be able to:**
- Explain the Time Value of Money (TVM) and why it matters for decisions
- Explain Present Value (PV), Future Value (FV), compounding and discounting
- Calculate the PV and FV of single and multiple cash flows
- Value annuities (ordinary and due) and perpetuities (constant and growing)
- Calculate and discount a Terminal Value
- Choose the right TVM technique for a given cash-flow pattern

---

## 1. The Big Picture: Where TVM Fits

The goal of corporate finance is to **maximize the value of the firm** through three decisions (Aswath Damodaran):

| Decision | Core question | Key idea |
|---|---|---|
| **Investment Decision** | Where to invest? | Invest only in assets that earn **more than the hurdle rate**. The hurdle rate reflects risk and the debt–equity mix; the return reflects the **size and timing** of cash flows |
| **Financing Decision** | How to fund it? | Find the right mix of debt and equity; the right debt **matches the tenor of your assets** |
| **Dividend Decision** | What to return to owners? | How much cash you return depends on current and future investment opportunities; how you return it (dividends or buybacks) depends on owners' preference |

> ⚠️ On the slide, the Dividend Decision box repeats the Financing Decision text ("Find the right kind of debt..."). The correct idea is: *if you cannot find investments that beat the hurdle rate, return the cash to the owners.*

"Timing of cash flows" is exactly where TVM comes in.

---

## 2. Why Money Has Time Value

### Three situations from class

| Situation | Choice | Best answer | Why |
|---|---|---|---|
| 1 | ₹10,000 today **or** ₹10,000 after 1 year | **Today** | Same amount, but today you can spend or invest it now |
| 2 | ₹10,000 today **or** ₹11,000 after 1 year, interest = 10% | **Indifferent** | ₹10,000 × 1.10 = ₹11,000, so both are equal |
| 3 | Same as 2, but the phone price rises 20% to ₹12,000 | **Today** | Buy the phone at ₹10,000 now. ₹11,000 next year can't buy a ₹12,000 phone (inflation beats interest) |

> ⚠️ Slides 4–5 say "the price of **chocolate** remains at ₹10,000". It should be the **phone**.

### Factors behind TVM
- **Consumption preference:** most people prefer consuming now over later.
- **Investment opportunity:** money today can be invested to earn a return.
- **Inflation:** purchasing power falls over time.
- **Opportunity cost:** every choice gives up the return on the next-best alternative.

> 💡 **In simple words:** Aaj ka ₹100 kal ke ₹100 se zyada valuable hai, kyunki aaj wala invest karke badh sakta hai, aur kal tak mehngai uski value kha jayegi.

---

## 3. Key Notations

| Symbol | Meaning |
|---|---|
| **PV** | Present Value: the value of a cash flow today |
| **FV** | Future Value: the value of a cash flow at a future date |
| **CF** | Cash Flow: an amount received or paid at a point in time |
| **r** | Interest / discount rate / required return per period |
| **n** (or t) | Number of compounding/discounting periods |
| **g** | Constant growth rate of cash flows |
| **A** | Annuity payment: the equal periodic cash flow |

---

## 4. Future Value & Compounding

Invest ₹100 at 10% per year:
- After 1 year: 100 × 1.10 = **₹110**
- After 2 years: 110 × 1.10 = **₹121** = 100 + 10 (Y1 interest) + 10 (Y2 interest) + 1 (**interest on interest**)
- After 3 years: 121 × 1.10 = **₹133.10**

$$\textbf{FV = PV × (1 + r)ⁿ}$$

(1 + r)ⁿ is the **Future Value Interest Factor (FVIF)**, also called the compounding factor.

> ⚠️ Slide 9 writes "Original principal ₹110 + Interest ₹10 + Interest on interest ₹1". The principal is ₹100; the ₹121 is 100 + 10 + 10 + 1. The formula there also uses "$1" instead of ₹1.

**Worked example:** ₹10,000 at 10% p.a. for 3 years → FV = 10,000 × 1.331 = **₹13,310**. The extra ₹3,310 comes purely from compounding.

### Simple vs Compound Interest (₹100 at 10%, 5 years)

| Year | Beginning (₹) | Simple Interest (₹) | Interest on Interest (₹) | Total Interest (₹) | Ending (₹) |
|---|---|---|---|---|---|
| 1 | 100.00 | 10.00 | 0.00 | 10.00 | 110.00 |
| 2 | 110.00 | 10.00 | 1.00 | 11.00 | 121.00 |
| 3 | 121.00 | 10.00 | 2.10 | 12.10 | 133.10 |
| 4 | 133.10 | 10.00 | 3.31 | 13.31 | 146.41 |
| 5 | 146.41 | 10.00 | 4.64 | 14.64 | 161.05 |
| **Total** | | **50.00** | **11.05** | **61.05** | |

Simple interest is earned only on the principal. Compound interest also earns on past interest, and the gap grows every year.

### The Power of Compounding
₹100 invested in 1900 at 10% for 120 years: 1.10¹²⁰ = 92,709.07, so FV = 100 × 92,709.07 = **₹92,70,906.88** (about ₹93 lakh).

> ⚠️ Slide 11 says ₹100 was invested but calculates with ₹10, giving ₹9,27,090.7. With ₹100, the answer is ₹92,70,906.88. "Comppounding" is also a typo.

### Monthly compounding: APR vs Effective Rate
A credit card charges **2% per month**. ₹100 of purchases after 1 year:
FV = 100 × (1.02)¹² = 100 × 1.2682 = **₹126.82**

| Measure | Calculation | Value |
|---|---|---|
| **Nominal APR** | 2% × 12 | **24%** |
| **Effective annual rate (EAR)** | (126.82 − 100) ÷ 100 | **26.82%** |

> ⚠️ The slide labels both numbers "APR". Strictly, **24% is the APR** (nominal) and **26.82% is the effective annual rate**, which is the true cost because of monthly compounding.

> 💡 **In simple words:** APR bas "2% × 12" hai. Asli kharcha EAR hai, kyunki har mahine interest pe bhi interest lagta hai.

---

## 5. Present Value & Discounting

Discounting is compounding in reverse: it brings a future amount back to today.

$$\textbf{PV = FV ÷ (1 + r)ⁿ}$$

1 ÷ (1 + r)ⁿ is the **discounting (PV) factor**.

| | Compounding | Discounting |
|---|---|---|
| Direction | PV → FV (forward in time) | FV → PV (backward in time) |
| Formula | FV = PV × (1 + r)ⁿ | PV = FV ÷ (1 + r)ⁿ |
| Factor | (1 + r)ⁿ | 1 ÷ (1 + r)ⁿ |

### Worked examples

| Question | Working | Answer |
|---|---|---|
| ₹10,000 at 7.5% for 5 years: FV? | 10,000 × (1.075)⁵ | **₹14,356.29** |
| Need ₹10,000 in 5 years at 7.5%: invest how much today? | 10,000 ÷ (1.075)⁵ | **₹6,965.59** |
| PV of ₹9,000 in 5 years at 16.5% | 9,000 ÷ (1.165)⁵ | **₹4,193.85** |
| PV of ₹1,200 in 12 years at 13% | 1,200 ÷ (1.13)¹² | **₹276.85** |
| PV of ₹50,000 in 5 years at 8% | 50,000 ÷ 1.4693 | **₹34,029** |

Check: 6,965.59 × (1.075)⁵ = 10,000 ✅. PV and FV are two views of the same value.

> ⚠️ Slide 19's algebra line writes "A = ₹1200/(1.13)**13**". It should be (1.13)**¹²**; with 13 the answer would be ₹245.00, not ₹276.85.

> 📌 **PV falls when r rises or n rises.** A higher rate or a longer wait means the future money is worth less today.

---

## 6. Multiple (Uneven) Cash Flows

When cash flows differ each period, discount each one with **its own t** and add them up:

$$\textbf{PV = Σ CFₜ ÷ (1 + r)ᵗ}$$

**Example:** A machine generates year-end cash flows; discount rate = 10%.

| Year | Cash Flow (₹) | PV Factor @10% | PV (₹) |
|---|---|---|---|
| 1 | 5,000 | 0.9091 | 4,545.45 |
| 2 | 6,000 | 0.8264 | 4,958.68 |
| 3 | 7,000 | 0.7513 | 5,259.20 |
| 4 | 8,000 | 0.6830 | 5,464.11 |
| **Total** | 26,000 | | **20,227.44** |

The uneven stream is worth **₹20,227.44 today**, much less than the ₹26,000 undiscounted total.

> ⚠️ The slide shows Year 3 PV = 5,259.95 and Year 4 PV = 5,464.23, giving a total of ₹20,228.31. The correct values are 7,000 ÷ 1.331 = **5,259.20** and 8,000 ÷ 1.4641 = **5,464.11**, so the total is **₹20,227.44**.

Used for: equipment cash-flow schedules, project appraisal, loans with irregular instalments.

---

## 7. Annuities: Equal Periodic Cash Flows

An **annuity** is a series of **equal** cash flows at **regular intervals** for a **fixed** number of periods.

| Type | When paid | Examples |
|---|---|---|
| **Ordinary annuity** | **End** of each period (t = 1 to n) | Loan EMIs, most bond coupons |
| **Annuity due** | **Start** of each period (t = 0 to n − 1) | Rent, insurance premiums paid in advance |

### Formulas (ordinary annuity)

| Value | Formula | Factor name |
|---|---|---|
| PV | A × [1 − (1 + r)⁻ⁿ] ÷ r | **PVIFA** |
| FV | A × [(1 + r)ⁿ − 1] ÷ r | **FVIFA** |

$$\textbf{Annuity Due = Ordinary Annuity value × (1 + r)}$$

Each payment of an annuity due arrives one period earlier, so its value is the ordinary value compounded forward by one period.

> 💡 **In simple words:** Annuity due mein har payment ek period pehle aata hai, isliye uski value hamesha ordinary annuity se (1 + r) guna zyada hoti hai.

### Example: ₹10,000 at the end of every year for 5 years at 10%

| | Working | Answer | Meaning |
|---|---|---|---|
| PV | 10,000 × 3.7908 | **₹37,908** | Lump sum today equal to the 5 deposits |
| FV | 10,000 × 6.1051 | **₹61,051** | Fund value at the end of year 5 |

Check: 37,908 × (1.10)⁵ = 37,908 × 1.61051 = ₹61,051 ✅. FV of an annuity = PV of the annuity compounded for n years.

If the same deposits were made at the **start** of each year (annuity due): PV = 37,908 × 1.10 = **₹41,699**; FV = 61,051 × 1.10 = **₹67,156**.

---

## 8. Perpetuities: Cash Flows Forever

A **perpetuity** is an annuity with **no end date**.

| Type | Formula | Condition |
|---|---|---|
| Constant (level) perpetuity | **PV = CF ÷ r** | |
| Growing perpetuity (Gordon model) | **PV = CF₁ ÷ (r − g)** | r > g |

**Examples (r = 10%):**
- Preference share paying ₹500 every year forever: PV = 500 ÷ 0.10 = **₹5,000**
- First payment ₹500, growing at 4% forever: PV = 500 ÷ (0.10 − 0.04) = **₹8,333**

> 📌 **CF₁ is next year's cash flow.** If you are given this year's payment (CF₀), first grow it: CF₁ = CF₀ × (1 + g).
> If g ≥ r, the formula breaks down (the value would be infinite or negative), so it applies only when r > g.

> 💡 **In simple words:** Perpetuity ka formula hai "har saal ka paisa ÷ rate". Growth ho to rate mein se growth ghatao, par growth hamesha rate se kam honi chahiye.

---

## 9. Terminal Value (TV)

In project or business valuation, we forecast cash flows **explicitly** for a few years. **Terminal Value** captures the value of **all cash flows after that**, assuming stable growth forever.

$$\textbf{TVₙ = [CFₙ × (1 + g)] ÷ (r − g)}$$
$$\textbf{PV of TV = TVₙ ÷ (1 + r)ⁿ}$$

- CFₙ = the last explicitly forecast cash flow (year n); g = long-term growth; r = discount rate
- TV is a value **at year n**, so it must be discounted back to today.

**Example:** Year 5 cash flow = ₹2,00,000, g = 3%, r = 12%.

| Step | Working | Answer |
|---|---|---|
| 1. TV at Year 5 | 2,00,000 × 1.03 ÷ (0.12 − 0.03) = 2,06,000 ÷ 0.09 | **₹22,88,889** |
| 2. PV of TV | 22,88,889 ÷ (1.12)⁵ = 22,88,889 ÷ 1.7623 | **₹12,98,808** |

> ⚠️ The slide shows PV of TV = ₹12,98,590. Dividing 22,88,889 by 1.7623 gives **₹12,98,808** (₹12,98,777 with the unrounded factor 1.76234).

Used in capital budgeting and in DCF valuation of projects, plants and businesses.

---

## 10. Summary: Which Formula When?

| Cash-flow type | Formula | Typical application |
|---|---|---|
| Single cash flow | FV = PV(1 + r)ⁿ / PV = FV ÷ (1 + r)ⁿ | Lump-sum purchase or resale value |
| Multiple cash flows | PV = Σ CFₜ ÷ (1 + r)ᵗ | Uneven project cash flows |
| Annuity | PV = A[1 − (1 + r)⁻ⁿ] ÷ r | Loan EMIs, AMC and lease payments |
| Perpetuity | PV = CF ÷ r (or CF₁ ÷ (r − g)) | Perpetual maintenance, endowment funds |
| Terminal value | TV = CFₙ(1 + g) ÷ (r − g) | Long-life project/plant value beyond the forecast |

---

## 11. Quick Revision 🧠

- **TVM:** ₹1 today > ₹1 tomorrow (consumption preference, investment opportunity, inflation, opportunity cost)
- **Compounding** (forward): FV = PV(1 + r)ⁿ · **Discounting** (backward): PV = FV ÷ (1 + r)ⁿ
- Compound interest = simple interest + **interest on interest**
- **APR** = periodic rate × periods; **EAR** = (1 + periodic rate)ᵖᵉʳⁱᵒᵈˢ − 1 (the true cost)
- Uneven flows: discount **each** with its own t, then add
- Annuity PV uses **PVIFA**, FV uses **FVIFA**; **Annuity due = ordinary × (1 + r)**
- Perpetuity: CF ÷ r; growing: CF₁ ÷ (r − g), only if r > g
- Terminal value is a year-n value: **always discount it back** by (1 + r)ⁿ
- Higher r or longer n → **lower PV**

---

## 12. MCQs (Mid–Advanced Level) 📝

**Q1.** You invest ₹25,000 at 8% p.a. compounded annually for 4 years. What is the future value?
- A) ₹34,012
- B) ₹31,250
- C) ₹33,000
- D) ₹36,733

> ✅ **Answer: A.** FV = 25,000 × (1.08)⁴ = 25,000 × 1.36049 = **₹34,012**. Option C (₹33,000) is simple interest: 25,000 + 4 × 2,000. 📖 Section 4

**Q2.** What is the present value of ₹1,00,000 receivable in 6 years if the discount rate is 9%?
- A) ₹59,627
- B) ₹1,67,710
- C) ₹91,743
- D) ₹46,000

> ✅ **Answer: A.** PV = 1,00,000 ÷ (1.09)⁶ = 1,00,000 ÷ 1.67710 = **₹59,627**. Option C discounts for only one year, and Option B is the future value factor applied instead. 📖 Section 5

**Q3.** A credit card charges 2% per month on the outstanding balance. What is the effective annual cost of borrowing?
- A) 24.00%
- B) 22.00%
- C) 26.82%
- D) 2.00%

> ✅ **Answer: C.** EAR = (1.02)¹² − 1 = **26.82%**. Option A (24%) is the nominal APR, which ignores monthly compounding. 📖 Section 4

**Q4.** A loan has a nominal rate of 12% per year, compounded monthly (1% per month). What is the effective annual rate?
- A) 12.00%
- B) 13.00%
- C) 12.36%
- D) 12.68%

> ✅ **Answer: D.** EAR = (1.01)¹² − 1 = **12.68%**. Option C (12.36%) is what semi-annual compounding (6% twice) would give. 📖 Section 4

**Q5.** ₹10,000 is invested for 2 years at 10% p.a. Interest is compounded **semi-annually**. What is the maturity value?
- A) ₹12,000
- B) ₹12,100
- C) ₹12,210
- D) ₹12,155

> ✅ **Answer: D.** Rate per half-year = 5%, periods = 4. FV = 10,000 × (1.05)⁴ = **₹12,155.06**. Option B is annual compounding, and Option A is simple interest. More frequent compounding gives a higher FV. 📖 Section 4

**Q6.** On ₹50,000 at 10% for 3 years, how much **more** does compound interest earn than simple interest?
- A) ₹1,550
- B) ₹16,550
- C) ₹1,000
- D) ₹5,000

> ✅ **Answer: A.** Simple interest = 50,000 × 10% × 3 = ₹15,000. Compound interest = 50,000 × (1.331 − 1) = ₹16,550. Difference = **₹1,550**, which is interest on interest. Option B is the total compound interest, not the difference. 📖 Section 4

**Q7.** A project gives ₹10,000, ₹20,000 and ₹30,000 at the end of Years 1, 2 and 3. At a 12% discount rate, what is the total PV?
- A) ₹53,571
- B) ₹46,226
- C) ₹42,707
- D) ₹60,000

> ✅ **Answer: B.** 10,000/1.12 + 20,000/1.2544 + 30,000/1.404928 = 8,928.57 + 15,943.88 + 21,353.41 = **₹46,225.86**. Option A discounts the total for one year only, and Option C discounts the whole ₹60,000 for three years. 📖 Section 6

**Q8.** What is the PV of an ordinary annuity of ₹10,000 per year for 3 years at 10%?
- A) ₹24,869
- B) ₹30,000
- C) ₹27,355
- D) ₹22,539

> ✅ **Answer: A.** PVIFA = [1 − (1.10)⁻³] ÷ 0.10 = 2.48685. PV = 10,000 × 2.48685 = **₹24,869**. Option C is the annuity-due value (× 1.10), and Option D wrongly discounts all ₹30,000 from Year 3. 📖 Section 7

**Q9.** A lease requires ₹20,000 at the **start** of each year for 5 years. The discount rate is 8%. What is the PV of the lease payments?
- A) ₹79,854
- B) ₹1,00,000
- C) ₹86,243
- D) ₹73,939

> ✅ **Answer: C.** Ordinary annuity PV = 20,000 × 3.99271 = ₹79,854. This is an annuity due, so PV = 79,854 × 1.08 = **₹86,243**. Option A treats the payments as end-of-year. 📖 Section 7

**Q10.** You deposit ₹5,000 at the end of every year for 10 years in a fund earning 12%. What is the fund value at the end of Year 10?
- A) ₹98,273
- B) ₹15,529
- C) ₹87,744
- D) ₹50,000

> ✅ **Answer: C.** FVIFA = [(1.12)¹⁰ − 1] ÷ 0.12 = 17.5487. FV = 5,000 × 17.5487 = **₹87,744**. Option A would be correct only for start-of-year deposits (annuity due), and Option B is the FV of a single ₹5,000. 📖 Section 7

**Q11.** A ₹5,00,000 loan at 10% p.a. is repaid in 5 equal **year-end** instalments. What is the annual instalment?
- A) ₹1,00,000
- B) ₹1,31,899
- C) ₹1,10,000
- D) ₹1,50,000

> ✅ **Answer: B.** Instalment = Loan ÷ PVIFA = 5,00,000 ÷ 3.79079 = **₹1,31,899**. Option A ignores interest, and Option D charges simple interest on the full principal for all 5 years. 📖 Section 7

**Q12.** A preference share pays a fixed dividend of ₹1,200 every year forever. If the required return is 8%, what is its value?
- A) ₹24,000
- B) ₹15,000
- C) ₹9,600
- D) ₹12,960

> ✅ **Answer: B.** Constant perpetuity: PV = 1,200 ÷ 0.08 = **₹15,000**. Option A would be the value only if the dividend grew at 3% per year (1,200 ÷ 0.05). 📖 Section 8

**Q13.** A company **just paid** a dividend of ₹100. Dividends will grow at 5% forever, and the required return is 12%. What is the share's value?
- A) ₹1,500.00
- B) ₹1,428.57
- C) ₹2,100.00
- D) ₹833.33

> ✅ **Answer: A.** The formula needs **next year's** cash flow: CF₁ = 100 × 1.05 = 105. PV = 105 ÷ (0.12 − 0.05) = **₹1,500**. Option B wrongly uses ₹100 instead of ₹105, and Option D ignores growth (100 ÷ 0.12). 📖 Section 8

**Q14.** A perpetuity of ₹1,000 per year will start at the end of **Year 4**. At 10%, what is it worth today?
- A) ₹10,000
- B) ₹6,830
- C) ₹7,513
- D) ₹9,091

> ✅ **Answer: C.** The perpetuity formula gives its value one period before the first payment: 1,000 ÷ 0.10 = ₹10,000 at the end of Year 3. Discount 3 years: 10,000 ÷ 1.331 = **₹7,513**. Option B discounts for 4 years, and Option A forgets to discount at all. 📖 Section 8

**Q15.** A project's explicit forecast ends in Year 3 with a cash flow of ₹50,000. After that, cash flows grow at 5% forever. The discount rate is 11%. What is the **present value** of the terminal value?
- A) ₹6,09,326
- B) ₹8,33,333
- C) ₹8,75,000
- D) ₹6,39,792

> ✅ **Answer: D.** TV₃ = 50,000 × 1.05 ÷ (0.11 − 0.05) = ₹8,75,000. PV = 8,75,000 ÷ (1.11)³ = 8,75,000 ÷ 1.367631 = **₹6,39,792**. Option C is the undiscounted TV, and Option B forgets to grow the Year 3 cash flow. 📖 Section 9

**Q16.** In the Gordon growth formula PV = CF₁ ÷ (r − g), what happens if the assumed growth rate is equal to or higher than the discount rate?
- A) The formula still works if cash flows are positive
- B) The value becomes zero
- C) The value equals CF₁
- D) The formula gives a meaningless (infinite or negative) value, so it cannot be used

> ✅ **Answer: D.** The denominator becomes zero or negative, so the result is infinite or negative. The model is valid only when r > g, which is why long-term g is usually kept close to the economy's growth rate. 📖 Section 8

**Q17.** You are offered ₹10,000 today or ₹11,000 one year from now. You can earn 12% on your money. Which should you choose?
- A) Either, because they are equal
- B) ₹10,000 today, because the PV of ₹11,000 is only about ₹9,821
- C) ₹11,000 next year, because it is a larger amount
- D) ₹11,000 next year, because its PV is ₹12,320

> ✅ **Answer: B.** PV of ₹11,000 = 11,000 ÷ 1.12 = **₹9,821**, which is less than ₹10,000. At 10% the two would be equal; at 12% today's money wins. Option D compounds instead of discounting. 📖 Section 2

**Q18.** Interest rates are 10%, but the price of the phone you want rises 20% in a year. Which TVM factor makes taking money today the better choice?
- A) Opportunity cost of time only
- B) The money measurement concept
- C) Inflation (loss of purchasing power) exceeding the interest earned
- D) Compounding frequency

> ✅ **Answer: C.** ₹10,000 grows to ₹11,000, but the phone now costs ₹12,000. Purchasing power fell faster than the money grew, so spending today is better. 📖 Section 2

**Q19.** Holding the future amount constant, which change **increases** its present value?
- A) A lower discount rate
- B) More frequent compounding at the same nominal rate
- C) A higher discount rate
- D) A longer time period

> ✅ **Answer: A.** PV = FV ÷ (1 + r)ⁿ. A lower r makes the denominator smaller, so PV rises. Option C, Option D and Option B all make the denominator larger and reduce PV. 📖 Section 5

**Q20.** Which statement about an annuity due versus an ordinary annuity (same A, r, n) is correct?
- A) Annuity due PV = ordinary annuity PV ÷ (1 + r)
- B) Annuity due PV = ordinary annuity PV × (1 + r)
- C) They always have the same PV
- D) Annuity due PV = ordinary annuity PV + A × n

> ✅ **Answer: B.** Every payment in an annuity due arrives one period earlier, so each is discounted one period less, which multiplies the whole value by (1 + r). The same relationship holds for FV. 📖 Section 7

**Q21.** Using the Rule of 72, roughly how long does money take to double at 9% p.a.?
- A) 6 years
- B) 9 years
- C) 11 years
- D) 8 years

> ✅ **Answer: D.** 72 ÷ 9 = **8 years**. Check: (1.09)⁸ = 1.993, almost exactly double (the exact answer is 8.04 years). This shortcut is not on the slides, but it is a quick check on compounding. 📖 Section 4
