# Session 8: Financial Planning & Analysis – 4 (Capital Budgeting Techniques)

> 🎓 Recap: Earlier sessions covered financial statements, ratios and cost concepts (incl. sunk and relevant costs). Today we answer the biggest question a CFO faces: **"Should we invest crores in this project or not?"**

**By the end of this session you will be able to:**
- Calculate the **Net Present Value (NPV)** of a project from its cash flows and discount rate
- Compute the **Internal Rate of Return (IRR)** by trial-and-error / interpolation
- Find the **Payback Period** and **Discounted Payback Period**
- Use the **Profitability Index (PI)** to allocate a limited capital budget (capital rationing)

---

## 1. What is Capital Budgeting?

**Capital Budgeting** is the process of planning and evaluating **large, long-term investments** (new machinery, a new product line, acquiring a business) whose cash flows extend many years into the future.

Why these decisions are special:
- **Large sums** are committed up front
- They are **largely irreversible** — once spent, the money is a sunk cost
- They shape the firm's **future cash flows, risk and competitiveness** for years or decades
- One wrong capital budgeting decision can destroy far more shareholder value than an operating mistake

### Types of investment decisions

| Type | Meaning | Example |
|---|---|---|
| **Expansion** | Adding new capacity to grow | Tata Motors' new EV plant at Sanand, Gujarat |
| **Replacement** | Replacing old/worn-out assets | Replacing a 15-year-old boiler with an efficient one |
| **Diversification** | Entering new products/markets to reduce risk | Reliance Industries entering retail |

### The process

**Identify opportunity → Estimate cash flows → Evaluate (NPV / IRR / Payback) → Decide (accept/reject) → Monitor & review**

> 💡 **In simple words:** Capital budgeting matlab bade, lambe investment ka "worth it hai ya nahi" test. Aaj ka paisa lagao, kal ke cash flows aayenge — kya wo kaafi hain?

---

## 2. Our Running Case: Bharat AutoParts Ltd. 🏭

An auto-components maker near Pune is considering a **robotic welding line**. We use this one case for every technique.

| Item | Value |
|---|---|
| Initial investment | **₹80,00,000** |
| Expected incremental cash flow | **₹25,00,000 per year for 5 years** |
| Cost of capital (discount rate) | **12% p.a.** |

---

## 3. Net Present Value (NPV) ⭐

NPV discounts every future cash flow back to today at the firm's **cost of capital**, then subtracts the initial investment. It tells you, **in today's rupees, how much value a project adds.**

$$NPV = -I_0 + \frac{CF_1}{(1+r)^1} + \frac{CF_2}{(1+r)^2} + \dots + \frac{CF_n}{(1+r)^n}$$

| Symbol | Meaning |
|---|---|
| I₀ | Initial investment |
| CF₁ … CFₙ | Future cash flows |
| r | Required rate of return (cost of capital) |
| n | Project life |

### Decision rule

| NPV | Decision | Why |
|---|---|---|
| **> 0** | Accept | Adds value beyond the required return |
| **= 0** | Indifferent | Earns exactly the required return |
| **< 0** | Reject | Destroys value |

**Why NPV is the gold standard:**
- It measures **rupees of value created** — exactly what shareholders care about
- NPVs of independent projects can be **added** (value-additivity principle)
- For **mutually exclusive** projects, NPV is the most reliable single criterion

> 💡 **In simple words:** ₹1 aaj ka > ₹1 kal ka. NPV saare future paise ko "aaj ke rupee" mein convert karke dekhta hai ki investment se zyada mila ya kam.

### Solved: Bharat AutoParts NPV

When cash flows are equal every year (an **annuity**), use the annuity factor:

$$PVIFA(r, n) = \frac{1-(1+r)^{-n}}{r}$$

| Step | Calculation | Result |
|---|---|---|
| PVIFA (12%, 5 yrs) | [1 − (1.12)⁻⁵] ÷ 0.12 | **3.6048** |
| PV of inflows | ₹25,00,000 × 3.6048 | **₹90,11,938** (approx.) |
| NPV | ₹90,11,938 − ₹80,00,000 | **₹10,11,938** |

✅ **Decision: ACCEPT.** Investing ₹80L today creates about **₹10.12L of extra value**, over and above the 12% return investors already require.

---

## 4. Internal Rate of Return (IRR)

IRR is the discount rate at which **NPV becomes exactly zero** — the project's own "break-even" rate of return.

$$0 = \sum \frac{CF_t}{(1+IRR)^t} - \text{Initial Investment}$$

**Decision rule:** Accept if **IRR > cost of capital**; reject if **IRR < cost of capital**.

It is found by trial-and-error, **interpolation**, or a spreadsheet `IRR()` function.

**Interpolation formula:**

$$IRR = L + \frac{NPV_L}{NPV_L - NPV_H} \times (H - L)$$

where L = lower rate, H = higher rate, NPV_L = NPV at L (positive), NPV_H = NPV at H (negative).

### Solved: Bharat AutoParts IRR (trial and error)

| Discount rate | PV of inflows (₹25L × 5 yrs) | NPV |
|---|---|---|
| 12% | ₹90,11,938 | + ₹10,11,938 |
| 17% | ₹79,98,365 | − ₹1,635 (≈ zero) |
| 18% | ₹78,17,928 | − ₹1,82,072 |

**IRR ≈ 17%** (exact ≈ 16.99%). Since 17% > 12%, **ACCEPT** — the same decision as NPV.

> ⚠️ The slide shows PV at 17% as ₹80,02,000 (NPV + ₹2,000). The correct figures are ₹79,98,365 and −₹1,635. Either way, IRR ≈ 17%, so the conclusion does not change.

> 💡 **In simple words:** NPV poochta hai "kitne rupaye bane?", IRR poochta hai "kitne percent return mila?". IRR wo rate hai jahan project na profit deta hai na loss — bilkul break-even.

---

## 5. When NPV and IRR Disagree 🏨

A hotel chain can fund only **one** of two **mutually exclusive** energy projects (cost of capital 10% for both):

| | Project A — Rooftop Solar | Project B — LED Retrofit |
|---|---|---|
| Investment | ₹1,00,00,000 | ₹10,00,000 |
| Cash flow | ₹28,00,000/yr for 6 yrs | ₹4,50,000/yr for 4 yrs |
| **NPV @ 10%** | **₹21,94,731** | ₹4,26,439 |
| **IRR** | 17.2% | **28.5%** |
| Ranked #1 by | NPV | IRR |

- B has the higher IRR, but **A creates far more total value** because it is much larger.
- NPV measures rupees of shareholder value, so **NPV wins when the rankings conflict**.
- **Rule of thumb:** for a single independent project, NPV or IRR both work. For **mutually exclusive** choices, **let NPV decide**.

### Why they disagree

| Reason | Explanation |
|---|---|
| **1. Scale differences** | IRR is a % and ignores investment size; a small project can show a huge IRR but add little value |
| **2. Reinvestment assumption** | IRR assumes cash flows are reinvested **at the IRR** (often unrealistic); NPV assumes reinvestment at the **cost of capital** |
| **3. Multiple IRRs** | Unconventional cash flows (e.g. a big decommissioning outflow at the end) can give **more than one IRR** |

> 💡 **In simple words:** ₹10 pe 50% return = ₹5. ₹1 crore pe 17% return = ₹17 lakh. Percentage bada hone se value badi nahi hoti — isliye mutually exclusive mein NPV dekho.

---

## 6. Payback Period

The time it takes for **cumulative cash flows to recover the initial investment** — a simple test of liquidity and risk.

$$\text{Payback} = \text{Years before recovery} + \frac{\text{Remaining amount}}{\text{Next year's cash flow}}$$

**Pros:** easy to explain to non-finance people; widely used by companies and startups.
**Cons:** ❌ ignores the time value of money; ❌ ignores all cash flows after the payback point.

### Solved: Bharat AutoParts Payback

| Year | Cash flow (₹) | Cumulative (₹) |
|---|---|---|
| 1 | 25,00,000 | 25,00,000 |
| 2 | 25,00,000 | 50,00,000 |
| 3 | 25,00,000 | 75,00,000 |
| 4 | 25,00,000 | 1,00,00,000 |

- ₹80L is recovered between Year 3 and Year 4.
- Still to recover after Year 3 = 80L − 75L = ₹5L
- Payback = 3 + (5L ÷ 25L) = **3.2 years** (≈ 3 years 2.4 months)

---

## 7. Discounted Payback Period

Fixes payback's biggest flaw: **discount each cash flow at the cost of capital first**, then accumulate.

### Solved: Bharat AutoParts @ 12%

| Year | Cash flow (₹) | Discounted CF @ 12% (₹) | Cumulative discounted CF (₹) |
|---|---|---|---|
| 1 | 25,00,000 | 22,32,143 | 22,32,143 |
| 2 | 25,00,000 | 19,92,985 | 42,25,128 |
| 3 | 25,00,000 | 17,79,451 | 60,04,579 |
| 4 | 25,00,000 | 15,88,795 | 75,93,374 |
| 5 | 25,00,000 | 14,18,567 | 90,11,941 |

- After Year 4, only ₹75,93,374 is recovered — short of ₹80L.
- Remaining = ₹4,06,626 ÷ Year-5 discounted CF ₹14,18,567 = 0.29
- **Discounted Payback ≈ 4.29 years**

✔️ Check: the Year-5 cumulative (₹90,11,941) equals the PV of inflows from the NPV section (rounding aside), and 90,11,941 − 80,00,000 ≈ NPV of ₹10.12L.

> ⚠️ The slide's discounted figures for Years 2–5 are slightly off (e.g. ₹19,93,000 instead of ₹19,92,985), so its Year-5 cumulative (₹90,11,992) doesn't match its own PV of ₹90,11,938. The answer (≈ 4.29 years) is unaffected.

> 📌 4.29 years (discounted) is much longer than 3.2 years (simple) — simple payback **understates** the true recovery time.

---

## 8. Comparison of Techniques

| Technique | Time value? | Key strengths | Key limitations |
|---|---|---|---|
| **NPV** | ✅ Yes | Measures ₹ value created; additive | Needs an accurate discount rate |
| **IRR** | ✅ Yes | Intuitive single % | Scale-blind; can conflict with NPV; multiple IRRs possible |
| **Payback** | ❌ No | Simple; shows liquidity risk | Ignores time value and post-payback cash flows |
| **Discounted Payback** | ✅ Yes | Fixes payback's time-value flaw | Still ignores cash flows after payback |

**Bharat AutoParts summary:** NPV ₹10.12L ✅ · IRR ≈ 17% > 12% ✅ · Payback 3.2 yrs · Discounted payback 4.29 yrs (within the 5-year life) ✅ → **Accept.**

---

## 9. Capital Rationing & the Profitability Index (PI)

In reality, funds are limited. When the budget is smaller than what all positive-NPV projects need, the firm must **ration capital**.

| Type | Meaning |
|---|---|
| **Soft rationing** | Self-imposed budget ceiling by management (planning/control) |
| **Hard rationing** | External constraint — lenders/markets won't supply more funds |

Under rationing, the highest-NPV project is not always best — look at **value per rupee invested**.

$$PI = \frac{\text{PV of future cash flows}}{\text{Initial investment}}$$

- **PI > 1 ⟺ NPV > 0** → accept
- For rationing: rank by PI (high → low) and fund down the list until the budget runs out

### Solved: Rationing a ₹1 crore budget (3 independent projects)

| Project | Investment (₹) | NPV (₹) | PI | PI rank |
|---|---|---|---|---|
| A — Solar Plant | 1,00,00,000 | 21,94,731 | 1.22 | 2 |
| B — LED Retrofit | 10,00,000 | 4,26,439 | 1.43 | 1 |
| C — Warehouse Automation | 40,00,000 | 5,48,960 | 1.14 | 3 |

- **By PI ranking:** fund B (₹10L) → A doesn't fit (needs ₹1 Cr) → fund C (₹40L). Used ₹50L, **total NPV = ₹9.75L**, ₹50L idle.
- **Fund A alone:** uses the full ₹1 Cr, **NPV = ₹21.95L** — more than B + C.
- ✅ **Choose A.**

> 📌 **Lesson:** PI ranking works for small, **divisible** projects. For large, **lumpy (indivisible)** projects, compare the **total NPV of every feasible combination**.

> 💡 **In simple words:** PI batata hai "har ₹1 lagane pe kitna PV wapas aaya". Par agar bada project tukdon mein nahi ho sakta, to combinations ka total NPV compare karo — wahi final judge hai.

---

## 10. Quick Revision 🧠

| Concept | Formula / Rule |
|---|---|
| NPV | Σ CFₜ/(1+r)ᵗ − I₀ → accept if > 0 |
| PVIFA | [1 − (1+r)⁻ⁿ] ÷ r (for equal annual cash flows) |
| IRR | Rate where NPV = 0 → accept if IRR > cost of capital |
| IRR interpolation | L + NPV_L/(NPV_L − NPV_H) × (H − L) |
| Payback | Years before recovery + remaining ÷ next year's CF |
| Discounted payback | Same, using discounted CFs (always longer than simple payback when r > 0) |
| PI | PV of inflows ÷ investment → accept if > 1 |
| Conflict | Mutually exclusive → NPV wins |
| Rationing | Rank by PI for divisible projects; compare total NPV of combinations for lumpy ones |

---

## 11. MCQs (Mid–Advanced Level) 📝

**Q1.** A project costs ₹50,00,000 and generates ₹15,00,000 per year for 5 years. The cost of capital is 10% (PVIFA = 3.7908). What is the NPV?
- A) ₹25,00,000
- B) −₹6,86,180
- C) ₹56,86,180
- D) ₹6,86,180

> ✅ **Answer: D.** PV of inflows = 15,00,000 × 3.7908 = ₹56,86,180. NPV = 56,86,180 − 50,00,000 = **₹6,86,180**. Option A ignores the time value of money (75L − 50L), and Option C is the PV of inflows, not the NPV. 📖 Section 3

**Q2.** A project has an NPV of exactly zero. What does this mean?
- A) The project earns exactly the required rate of return
- B) The project should always be rejected
- C) The project's IRR is zero
- D) The project makes no accounting profit

> ✅ **Answer: A.** NPV = 0 means the project earns exactly the cost of capital, so the firm is indifferent. It also means IRR equals the cost of capital, not zero (Option C). 📖 Section 3

**Q3.** A project costs ₹60L and its cash flows are ₹20L, ₹25L, ₹30L and ₹10L in Years 1–4. What is the simple payback period?
- A) 2.5 years
- B) 2.0 years
- C) 3.0 years
- D) 3.5 years

> ✅ **Answer: A.** Cumulative: 20L, 45L, 75L. After Year 2, ₹15L remains. 15 ÷ 30 = 0.5, so payback = **2.5 years**. 📖 Section 6

**Q4.** At 15% the NPV of a project is +₹4L, and at 20% it is −₹6L. Using interpolation, what is the IRR?
- A) 16%
- B) 17.5%
- C) 17%
- D) 18%

> ✅ **Answer: C.** IRR = 15 + [4 ÷ (4 − (−6))] × 5 = 15 + 0.4 × 5 = **17%**. 📖 Section 4

**Q5.** Project X has an NPV of ₹12L and an IRR of 18%. Project Y has an NPV of ₹8L and an IRR of 25%. They are mutually exclusive. Which should be chosen?
- A) Project X, because it has the higher NPV
- B) Project Y, because it probably has the shorter payback
- C) Neither, because the rankings conflict
- D) Project Y, because it has the higher IRR

> ✅ **Answer: A.** For mutually exclusive projects, NPV decides because it measures the actual rupee value created. Option D falls into the scale trap. 📖 Section 5

**Q6.** A project's PV of future cash flows is ₹1.3 Cr and its initial investment is ₹1 Cr. What are its PI and NPV?
- A) PI 1.30; NPV ₹1.3 Cr
- B) PI 0.30; NPV ₹30L
- C) PI 0.77; NPV −₹30L
- D) PI 1.30; NPV ₹30L

> ✅ **Answer: D.** PI = 1.3 ÷ 1 = **1.30**. NPV = 1.3 Cr − 1 Cr = **₹30L**. Option C inverts the ratio, and Option B computes NPV ÷ investment instead of PI. 📖 Section 9

**Q7.** A project has a Profitability Index of 0.90. What can you conclude?
- A) Payback period is less than one year
- B) NPV is positive, so accept the project
- C) IRR is higher than the cost of capital
- D) NPV is negative, so reject the project

> ✅ **Answer: D.** PI < 1 means the PV of inflows is less than the investment, so NPV < 0 and the IRR is below the cost of capital. 📖 Section 9

**Q8.** What does IRR assume about the reinvestment of intermediate cash flows?
- A) They are reinvested at the cost of capital
- B) They are reinvested at the IRR itself
- C) They are reinvested at the risk-free rate
- D) They are not reinvested at all

> ✅ **Answer: B.** IRR implicitly assumes reinvestment at the IRR, which is often unrealistic for high-IRR projects. NPV assumes reinvestment at the cost of capital (Option A). 📖 Section 5

**Q9.** A mining project has an outflow today, inflows for 8 years, and a large clean-up outflow in Year 9. Which problem is most likely?
- A) Negative payback period
- B) Multiple IRRs
- C) PI will always be exactly 1
- D) NPV cannot be calculated

> ✅ **Answer: B.** Unconventional cash flows (the sign changes more than once) can give more than one IRR, making the IRR rule ambiguous. NPV still works fine, so Option D is wrong. 📖 Section 5

**Q10.** Which statement about payback period is correct?
- A) It considers all cash flows over the project's life
- B) It ignores the time value of money and cash flows after the payback point
- C) It always gives the same ranking as NPV
- D) It is the best measure of value created for shareholders

> ✅ **Answer: B.** Simple payback treats every rupee as equal regardless of timing and stops counting once the investment is recovered. 📖 Section 6

**Q11.** A project costs ₹10L and generates ₹5L per year for 3 years. The cost of capital is 10%. What is the discounted payback period (approximately)?
- A) 2.00 years
- B) 2.35 years
- C) 3.00 years
- D) 2.50 years

> ✅ **Answer: B.** Discounted CFs: 4.55L, 4.13L, 3.76L. Cumulative after Year 2 = 8.68L. Remaining 1.32L ÷ 3.76L = 0.35, so ≈ **2.35 years**. Option A is the simple payback (10 ÷ 5). 📖 Section 7

**Q12.** For a project with positive cash flows and a positive discount rate, how does the discounted payback period compare with the simple payback period?
- A) It is always longer
- B) They are always equal
- C) It is always shorter
- D) It can be longer or shorter depending on the project

> ✅ **Answer: A.** Discounting reduces every future cash flow, so it takes longer to recover the investment. Bharat AutoParts: 4.29 years vs 3.2 years. 📖 Section 7

**Q13.** Project P has an NPV of ₹5L and Project Q has an NPV of ₹3L. They are independent and both are undertaken. What is the combined NPV?
- A) ₹8L
- B) ₹4L
- C) It cannot be determined without the IRRs
- D) ₹5L

> ✅ **Answer: A.** NPV follows the **value-additivity principle**: independent NPVs simply add up. IRRs, by contrast, cannot be added. 📖 Section 3

**Q14.** A company's budget is ₹60L. Project P needs ₹60L (NPV ₹15L), Q needs ₹30L (NPV ₹9L) and R needs ₹30L (NPV ₹7L). All are indivisible and independent. What should the company fund?
- A) P and Q
- B) Q and R
- C) P alone
- D) Q alone

> ✅ **Answer: B.** Feasible combinations within ₹60L: P alone (NPV 15L) or Q + R (NPV 16L). Q + R gives the higher total NPV. Option A exceeds the budget (₹90L). 📖 Section 9

**Q15.** Management caps next year's capital spending at ₹20 Cr to keep control over expansion, even though banks are willing to lend more. This is an example of:
- A) Multiple IRRs
- B) Hard capital rationing
- C) Soft capital rationing
- D) A sunk cost

> ✅ **Answer: C.** A self-imposed ceiling set by management is soft rationing. Hard rationing (Option B) comes from external limits, such as lenders refusing to lend. 📖 Section 9

**Q16.** If a firm's cost of capital rises from 10% to 14%, what happens to a conventional project's NPV and IRR?
- A) NPV stays the same; IRR rises
- B) Both fall
- C) NPV falls; IRR stays the same
- D) NPV rises; IRR falls

> ✅ **Answer: C.** A higher discount rate lowers the PV of future inflows, so NPV falls. IRR depends only on the project's own cash flows, so it doesn't change. 📖 Sections 3 & 4

**Q17.** You invest ₹1,00,000 today and receive ₹1,21,000 at the end of Year 2, with nothing in between. What is the IRR?
- A) 21%
- B) 11%
- C) 10.5%
- D) 10%

> ✅ **Answer: D.** 1,00,000 × (1 + IRR)² = 1,21,000, so (1 + IRR)² = 1.21 and IRR = **10%**. Option A is the total 2-year return, and Option C is the simple average (21% ÷ 2). 📖 Section 4

**Q18.** The PV of an annuity of ₹10L per year for 3 years at 10% is closest to:
- A) ₹22.60L
- B) ₹24.87L
- C) ₹27.27L
- D) ₹30.00L

> ✅ **Answer: B.** PVIFA(10%, 3) = 2.4869, so PV = 10L × 2.4869 ≈ **₹24.87L**. Option D ignores discounting. 📖 Section 3

**Q19.** A company spent ₹10L last year on a feasibility study for a new plant. While calculating the plant's NPV today, how should this ₹10L be treated?
- A) Deduct it from Year-1 cash flow
- B) Spread it over the project's life as depreciation
- C) Add it to the initial investment
- D) Ignore it, because it is a sunk cost

> ✅ **Answer: D.** Only future, incremental cash flows matter. The study cost is already spent and won't change with the decision, so it is a sunk cost (see Session 5). 📖 Section 1

**Q20.** A project's IRR is 14% and the firm's cost of capital is 15%. What is the correct decision?
- A) Reject, because payback must be too long
- B) Accept, because 14% is close to 15%
- C) Reject, because NPV at 15% will be negative
- D) Accept, because IRR is positive

> ✅ **Answer: C.** IRR < cost of capital means the project earns less than investors require, so NPV at 15% is negative. A positive IRR (Option D) is not enough. 📖 Section 4

**Q21.** In the hotel chain case, Project B (LED) has an IRR of 28.5% and Project A (Solar) has 17.2%, yet Project A is chosen. Why?
- A) Project A creates more total value (higher NPV) because of its much larger scale
- B) Project A's cash flows are reinvested at 28.5%
- C) Project A has the shorter payback period
- D) Project B has multiple IRRs

> ✅ **Answer: A.** NPV A = ₹21.95L vs NPV B = ₹4.26L. IRR is scale-blind; for mutually exclusive choices, the higher NPV wins. 📖 Section 5

**Q22.** Two projects both have a payback period of 3 years. Project M has no cash flows after Year 3, while Project N earns ₹20L per year for 5 more years. What does this show?
- A) M is better because it finishes sooner
- B) M and N create the same value
- C) Payback ignores cash flows after the payback point, so it cannot tell M and N apart
- D) Payback includes all cash flows, so N must have a shorter payback

> ✅ **Answer: C.** Payback stops counting once the investment is recovered, so it misses N's extra ₹1 Cr of inflows. NPV would clearly favour N. 📖 Section 6
