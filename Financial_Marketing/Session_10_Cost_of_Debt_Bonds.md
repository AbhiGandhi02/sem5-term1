# Session 10: Cost of Debt – Bonds & Stock Valuation

> 🎓 So far we have built financial statements, analysed ratios, and studied costing and planning. Today we move into **valuation**: how much is a bond or a share really worth today? The core idea is simple: **value = present value of the future cash flows you will receive.**

**By the end of this session you will be able to:**
- Explain the features of debt as a source of finance
- Define a bond and its key terms (face value, coupon, redemption, YTM)
- Price coupon, zero-coupon and perpetual bonds
- Estimate Yield to Maturity (YTM) with the approximation formula
- Value shares using the Dividend Discount Model (zero growth, constant growth, two-stage growth)

> ⚠️ The file is titled "Cost of Debt – Bonds", but the deck itself is a **"Bonds & Stocks – Valuation Module"**. It covers bond pricing and YTM, and also equity valuation (DDM). The slides say all figures are illustrative.

---

## 1. Features of Debt as a Source of Finance

| # | Feature | Meaning |
|---|---|---|
| 1 | **Contractual obligation** | The issuer must pay interest and repay the principal **whether or not it makes a profit** |
| 2 | **Fixed periodic payments** | The coupon is pre-set, unlike equity dividends, which are discretionary |
| 3 | **Priority over equity** | Debt holders are paid **before** shareholders, even on liquidation |
| 4 | **No ownership / voting** | Bondholders are **creditors**, not owners |
| 5 | **Finite maturity (usually)** | Most debt has a fixed repayment date (perpetual bonds are the exception) |
| 6 | **Tax-deductible interest** | Interest reduces taxable income, which is the **tax shield** on debt |

**Slide illustration:** Reliance Industries' issue of secured NCDs (Non-Convertible Debentures, face value ₹1,00,000 each) created a fixed obligation to pay interest and repay principal by the maturity date, without giving investors any ownership.

> 💡 **In simple words:** Debt = udhaar. Profit ho ya na ho, byaaj aur principal dena hi padega. Badle mein lender ko company mein hissa ya vote nahi milta, par paisa wapas pehle milta hai. Aur byaaj pe tax bachta hai, isliye debt "sasta" padta hai.

---

## 2. What is a Bond?

A **bond** is a debt security issued by a company, bank, PSU or the Government of India to raise long-term funds. The issuer promises to:
1. Pay a fixed periodic **coupon** (interest), and
2. Repay the **face value** (principal) on the **maturity / redemption date**.

Think of it as an **IOU**: you lend ₹1,000 today, the issuer pays you fixed interest and returns your ₹1,000 at maturity.

| Issuer | Examples |
|---|---|
| Government of India | G-Secs, Treasury Bills |
| PSUs | NHAI, REC, PFC, IRFC |
| Banks | SBI, HDFC Bank, ICICI (AT1 / Tier-2 bonds) |
| Corporates | Reliance, Tata Steel, M&M (NCDs) |

### Key Bond Terminology

| Term | Meaning |
|---|---|
| **Nominal / Face Value (F)** | Value printed on the bond (₹100 / ₹1,000 / ₹1,00,000). Also the amount repaid at redemption |
| **Coupon Rate** | Stated annual interest rate applied to face value, e.g. 8% on ₹1,000 |
| **Coupon (C)** | Actual rupee interest per period. **C = Coupon Rate × Face Value** (8% × 1,000 = ₹80) |
| **Redemption / Maturity Value** | Amount paid at maturity, usually equal to face value (redeemed **at par**) |
| **Maturity Date** | Date on which redemption happens and the bond ends |
| **Market Price** | Current traded price. Can be at a **discount**, **premium** or **par** to face value |
| **Yield to Maturity (YTM)** | Total return earned if you buy at the market price and hold till redemption |

> 💡 Coupon rate is fixed on the bond forever. The **required yield (r)** changes with market interest rates. The price adjusts so that the buyer earns the market yield.

---

## 3. Coupon (Plain Vanilla) Bond

Pays a fixed coupon every year and returns the full face value at redemption.

$$P = \frac{C}{(1+r)^1} + \frac{C}{(1+r)^2} + \dots + \frac{C + F}{(1+r)^T}$$

Price = PV of all coupons + PV of the face value repaid at maturity.

| Relationship | Bond trades at | Why |
|---|---|---|
| Coupon rate **<** required yield (YTM) | **Discount** (P < F) | The coupon is less attractive than the market, so the price must fall |
| Coupon rate **>** required yield | **Premium** (P > F) | The coupon is better than the market, so buyers pay extra |
| Coupon rate **=** required yield | **Par** (P = F) | No adjustment needed |

**Redemption is fixed:** whatever price you pay, you receive the full face value at maturity.

> 💡 **In simple words:** Bond price aur interest rate ulta chalte hain (see-saw). Market rate badha → purana fixed coupon kam attractive → price gira. Market rate gira → price badha.

### Solved Examples

**Example 1: Tata Steel bond (discount).** F = ₹1,000, coupon 8% (C = ₹80), T = 3 years, r = 9%.

| Year | Cash flow | PV @ 9% |
|---|---|---|
| 1 | 80 | 73.39 |
| 2 | 80 | 67.33 |
| 3 | 1,080 | 833.96 |
| **Price** | | **≈ ₹974.69** |

Coupon (8%) < YTM (9%), so it trades at a **discount**. ₹1,000 is still repaid at redemption.

> ⚠️ The slide shows Year 3 PV as 833.83 and the price as **₹974.56**. Precisely, 1,080 / 1.09³ = 833.96, so the price is **≈ ₹974.69**. The conclusion (discount) is unchanged.

**Example 2: Mahindra & Mahindra bond (premium).** F = ₹1,000, coupon 10% (C = ₹100), T = 2 years, r = 8%.

P = 100/1.08 + 1,100/1.08² = 92.59 + 943.07 = **₹1,035.66**

Coupon (10%) > YTM (8%), so it trades at a **premium**. Redemption is still ₹1,000.

---

## 4. Zero-Coupon Bond

Pays **no** periodic interest. Bought at a **deep discount**; the entire return is the gap between the purchase price and the face value received at redemption.

$$P = \frac{F}{(1+r)^T}$$

| Feature | Meaning |
|---|---|
| **No coupon rate** | No periodic interest at all |
| **No reinvestment risk** | There are no coupons to reinvest (unlike a coupon bond) |
| **Full return at redemption** | Return = Redemption value − Purchase price |

> 💡 **Reinvestment risk** = the risk that coupons you receive can only be reinvested at a lower rate than expected. A zero-coupon bond avoids it because it pays nothing until the end.

### Solved Examples

| | NHAI Zero-Coupon | REC Ltd Zero-Coupon |
|---|---|---|
| Face value (F) | ₹1,00,000 | ₹10,000 |
| Required yield (r) | 7% | 8% |
| Maturity (T) | 5 years | 10 years |
| Discount factor (1+r)ᵀ | 1.07⁵ = 1.4026 | 1.08¹⁰ = 2.1589 |
| **Price** | 1,00,000 / 1.4026 ≈ **₹71,299** | 10,000 / 2.1589 ≈ **₹4,632** |
| Investor gain | ≈ ₹28,701 after 5 years | ≈ ₹5,368 after 10 years |

The REC example shows why zero-coupon bonds suit **goal-based saving**: pay ₹4,632 now, receive ₹10,000 in 10 years, with nothing to reinvest in between.

---

## 5. Perpetual Bond

No maturity date. The issuer pays the coupon **forever** (subject to a call option), and the face value is **never redeemed**.

$$P = \frac{C}{r}$$

### In India: Additional Tier-1 (AT1) Bonds
- Issued by banks such as SBI, HDFC Bank, ICICI Bank and PNB under **Basel III** norms.
- AT1 bonds rank **just above equity** and can be **written down** under regulatory stress.
- Example: Yes Bank's AT1 bonds (~₹8,400 crore) were written off in 2020 on RBI's direction.

> 📌 **Lesson:** A bond's safety depends on **where it sits in the capital structure**, not just on being called a "bond".

### Solved Examples

| | Bank of Baroda AT1 | PFC Perpetual |
|---|---|---|
| Face value | ₹100 (never redeemed) | ₹100 (never redeemed) |
| Coupon | 8% → ₹8/yr | 9% → ₹9/yr |
| Required yield | 8.5% | 9.5% |
| **Price** | 8 / 0.085 ≈ **₹94.12** | 9 / 0.095 ≈ **₹94.74** |

Face value plays **no role** in the formula. Only the coupon and the required yield matter. In both cases coupon < required yield, so the price is below ₹100.

---

## 6. Yield to Maturity (YTM)

**YTM** = total return if you buy the bond at today's market price and hold it until redemption. It combines **coupon income** and any **capital gain/loss** versus face value.

**Approximation formula (for hand calculation):**

$$YTM \approx \frac{C + \dfrac{F - P}{n}}{\dfrac{F + P}{2}}$$

- Numerator = annual coupon + capital gain (or loss) spread over the years left
- Denominator = average of face value and price (average investment)

| Price vs face value | YTM vs coupon rate | Why |
|---|---|---|
| Price **below** face | YTM **>** coupon rate | You gain on redemption |
| Price **above** face | YTM **<** coupon rate | You lose a little on redemption |

> 💡 **In simple words:** Coupon rate sirf "printed" rate hai. YTM tumhara asli return hai, jo is baat pe depend karta hai ki tumne bond kitne mein kharida. Sasta kharida → YTM zyada.

### Solved Examples

| | Infosys-style bond | L&T-style bond |
|---|---|---|
| F | ₹1,000 | ₹1,000 |
| Coupon | 9% → ₹90 | 7% → ₹70 |
| n | 2 years | 3 years |
| Price (P) | ₹980 | ₹950 |
| Numerator | 90 + 20/2 = 100 | 70 + 50/3 = 86.67 |
| Denominator | (1,000 + 980)/2 = 990 | (1,000 + 950)/2 = 975 |
| **Approx. YTM** | **≈ 10.10%** | **≈ 8.89%** |
| Exact (IRR) YTM | ≈ 10.15% | ≈ 8.97% |

Both were bought below face value, so YTM > coupon rate.

> ⚠️ The slide calls 10.10% the "true YTM". It is the **approximate** YTM; the exact IRR-based YTM is ≈ 10.15%. The approximation is close but not exact.

> 📌 **Link to cost of debt:** the YTM is the return the market demands from the company's debt, so it is the company's **pre-tax cost of debt**. Because interest is tax-deductible (Section 1), the effective cost is lower.

---

## 7. Stock (Equity) Valuation: Dividend Discount Model (DDM)

For shares, **dividends play the role of coupons**, but equity has **no redemption value**. Price = PV of all expected future dividends.

### 7A. Zero Growth Model
Dividend is constant forever (like a perpetual bond). Suits mature, no-growth firms.

$$P_0 = \frac{D_1}{r}$$

| | Coal India | NTPC |
|---|---|---|
| D₁ (constant) | ₹18 | ₹9 |
| Required return (r) | 12% | 10% |
| **Price** | 18 / 0.12 = **₹150** | 9 / 0.10 = **₹90** |
| Why it fits | Stable, high-payout PSU | Mature utility, flat dividends |

### 7B. Constant Growth (Gordon Growth) Model
Dividends grow at a steady rate **g** forever. Suits stable companies growing at a modest, sustainable rate.

$$P_0 = \frac{D_1}{r - g}, \qquad D_1 = D_0(1 + g)$$

| | HDFC Bank | Asian Paints |
|---|---|---|
| Last dividend D₀ | ₹19 | ₹15 |
| g | 10% | 8% |
| r | 14% | 12% |
| D₁ | 19 × 1.10 = ₹20.90 | 15 × 1.08 = ₹16.20 |
| **Price** | 20.90 / 0.04 = **₹522.50** | 16.20 / 0.04 = **₹405.00** |

> ⚠️ Only valid when **r > g**. If g ≥ r, the formula gives an infinite or negative price, so it breaks down for companies in a high-growth phase.

> 💡 **D₀ vs D₁:** D₀ = dividend **just paid**. D₁ = **next year's** expected dividend. Gordon always uses D₁, so first grow D₀ by (1 + g). Using D₀ directly is the most common exam mistake.

### 7C. Two-Stage Growth Model
For companies (e.g. Indian IT/consumer firms) expected to out-grow the economy for a few years and then mature.

| Stage | Growth |
|---|---|
| **Stage 1: High growth** | Dividends grow fast at **g₁** for **n** years |
| **Stage 2: Stable growth** | Growth settles at a long-run rate **g₂** forever |

$$P_0 = \underbrace{\sum_{t=1}^{n} \frac{D_t}{(1+r)^t}}_{\text{PV of high-growth dividends}} + \underbrace{\frac{P_n}{(1+r)^n}}_{\text{PV of terminal price}}, \qquad P_n = \frac{D_{n+1}}{r - g_2}$$

The slide also gives the closed form for stage 1: D₁[1 − ((1+g₁)/(1+r))ⁿ] / (r − g₁). It gives the same answer as adding the PVs one by one.

> ⚠️ The slide writes the terminal term as Pₙ/(1+r)**ᵗ**. The exponent should be **n** (the terminal price is at the end of year n).

> 💡 **In simple words:** Pehle kuch saal tez growth wale dividends ko alag-alag discount karo. Phir saal n ke end pe company "normal" ho jaati hai, to wahan Gordon laga ke ek terminal price nikaalo aur use bhi aaj tak discount karo. Dono jodo.

**Example 1:** D₀ = ₹20, g₁ = 15% for 3 years, g₂ = 6%, r = 13%.

| Step | Working | Value |
|---|---|---|
| 1. Dividends | D₁ = 23.00, D₂ = 26.45, D₃ = 30.42 | |
| 2. PV @ 13% | 20.35 + 20.72 + 21.08 | **₹62.15** |
| 3. Terminal price | D₄ = 30.42 × 1.06 = 32.24; P₃ = 32.24 / 0.07 | ₹460.61 |
| 4. PV of P₃ | 460.61 / 1.13³ | **₹319.22** |
| **P₀** | 62.15 + 319.22 | **₹381.37** |

**Example 2:** D₀ = ₹8, g₁ = 20% for 2 years, g₂ = 7%, r = 15%.

| Step | Working | Value |
|---|---|---|
| 1. Dividends | D₁ = 9.60, D₂ = 11.52 | |
| 2. PV @ 15% | 8.35 + 8.71 | **₹17.06** |
| 3. Terminal price | D₃ = 11.52 × 1.07 = 12.33; P₂ = 12.33 / 0.08 | ₹154.08 |
| 4. PV of P₂ | 154.08 / 1.15² | **₹116.51** |
| **P₀** | 17.06 + 116.51 | **₹133.57** |

> Note: 12.33 / 0.08 = 154.13; the slide's ₹154.08 uses the unrounded D₃ (12.3264). The difference is negligible.

---

## 8. Quick Revision 🧠

| Concept | Formula | Remember |
|---|---|---|
| Coupon bond | P = Σ C/(1+r)ᵗ + F/(1+r)ᵀ | Coupon < yield → discount; coupon > yield → premium |
| Zero-coupon bond | P = F/(1+r)ᵀ | No coupon, no reinvestment risk |
| Perpetual bond | P = C/r | Face value never redeemed; AT1 bonds |
| YTM (approx.) | [C + (F−P)/n] / [(F+P)/2] | Price < F → YTM > coupon |
| Zero growth stock | P₀ = D₁/r | Constant dividend forever |
| Gordon growth | P₀ = D₁/(r−g), D₁ = D₀(1+g) | Needs r > g |
| Two-stage growth | PV(stage-1 dividends) + PV(Pₙ) | Pₙ = Dₙ₊₁/(r−g₂) |

- **Bonds:** fixed, contractual cash flows (coupon + redemption). **Stocks:** growth-linked, uncertain dividends, no redemption.
- Bond prices and interest rates move in **opposite** directions.
- Debt is cheaper partly because interest is **tax-deductible**.

---

## 9. MCQs (Mid–Advanced Level) 📝

**Q1.** A bond has a face value of ₹1,000, a 6% annual coupon and 2 years to maturity. If the required yield is 8%, what is its price?
- A) ₹1,037.04
- B) ₹1,000.00
- C) ₹964.33
- D) ₹943.40

> ✅ **Answer: C.** P = 60/1.08 + 1,060/1.08² = 55.56 + 908.78 = **₹964.33**. Option B discounts at the coupon rate (6%) instead of the required yield. Option A discounts both cash flows by only one year. 📖 Section 3

**Q2.** A bond carries a 9% coupon while similar bonds in the market yield 7%. Where will it trade?
- A) At a discount, because the market yield is lower
- B) At a discount, because its coupon is higher than the market yield
- C) At a premium, because its coupon is higher than the market yield
- D) At par, because redemption is always at face value

> ✅ **Answer: C.** When coupon rate > required yield, buyers pay more than face value to get the higher coupon, so the bond trades at a **premium**. 📖 Section 3

**Q3.** Market interest rates rise from 8% to 10%. What happens to a 3-year, 8% coupon bond (F = ₹1,000) that was trading at par?
- A) Price falls to about ₹950.26
- B) Price rises to about ₹1,050
- C) Coupon rises to 10%
- D) Price stays at ₹1,000 because the coupon is fixed

> ✅ **Answer: A.** At r = 10%: 80/1.1 + 80/1.1² + 1,080/1.1³ = **₹950.26**. Bond prices move opposite to interest rates; the coupon itself never changes. 📖 Section 3

**Q4.** A zero-coupon bond with face value ₹50,000 matures in 4 years. If the required yield is 6%, what is its price?
- A) ₹63,123.85
- B) ₹40,322.58
- C) ₹47,000.00
- D) ₹39,604.68

> ✅ **Answer: D.** P = 50,000 / 1.06⁴ = 50,000 / 1.26248 = **₹39,604.68**. Option B uses simple interest (1 + 0.06 × 4), and Option A compounds forward (future value) instead of discounting. 📖 Section 4

**Q5.** An investor buys a zero-coupon bond for ₹7,350 that will pay ₹10,000 in 4 years. What is the approximate annual yield?
- A) 8.0%
- B) 6.0%
- C) 9.0%
- D) 36.1%

> ✅ **Answer: A.** (10,000 / 7,350)^(1/4) − 1 ≈ **8.0%**. Option C divides the total gain by 4 years (simple interest), and Option D is the total 4-year return, not an annual yield. 📖 Section 4

**Q6.** Which statement about a zero-coupon bond is correct?
- A) It has no reinvestment risk because there are no coupons to reinvest
- B) It pays a small coupon at maturity only
- C) It always trades at a premium to face value
- D) It has no maturity date

> ✅ **Answer: A.** No periodic coupons means nothing to reinvest, so there is no reinvestment risk. It is bought at a deep discount, and it does have a maturity date. 📖 Section 4

**Q7.** A perpetual bond pays a ₹10 coupon every year forever. If the required yield is 12.5%, what is its price?
- A) ₹12.50
- B) ₹125
- C) ₹100
- D) ₹80

> ✅ **Answer: D.** P = C / r = 10 / 0.125 = **₹80**. The face value plays no role in the formula. 📖 Section 5

**Q8.** A perpetual bond pays ₹8 a year. If the required yield rises from 8% to 10%, by how much does its price change?
- A) Rises from ₹80 to ₹100
- B) Falls from ₹100 to ₹80, i.e. 20%
- C) No change, because it is never redeemed
- D) Falls by 2%

> ✅ **Answer: B.** 8/0.08 = ₹100 and 8/0.10 = ₹80, a 20% fall. Perpetual bonds are very sensitive to interest rates because all value comes from coupons far into the future. 📖 Section 5

**Q9.** What is the key lesson from the write-down of Yes Bank's AT1 bonds in 2020?
- A) A bond's safety depends on where it sits in the capital structure
- B) All bonds are risk-free because they are debt
- C) AT1 bonds rank above secured debt
- D) Perpetual bonds are always redeemed at face value

> ✅ **Answer: A.** AT1 bonds rank just above equity and can be written down under regulatory stress. Being called a "bond" does not make an instrument safe. 📖 Section 5

**Q10.** A bond has F = ₹1,000, an 8% coupon, 4 years to maturity and a market price of ₹920. What is its approximate YTM?
- A) 8.00%
- B) 10.42%
- C) 16.67%
- D) 8.70%

> ✅ **Answer: B.** [80 + (1,000 − 920)/4] / [(1,000 + 920)/2] = 100 / 960 = **10.42%**. Option D is the current yield (80/920), which ignores the capital gain, and Option C adds the whole ₹80 gain in one year. 📖 Section 6

**Q11.** A bond with a 7% coupon trades at ₹1,040 (face value ₹1,000). Which is true?
- A) YTM < 7%
- B) YTM = 7%
- C) YTM cannot be compared with the coupon rate
- D) YTM > 7%

> ✅ **Answer: A.** Bought above face value, the investor loses ₹40 at redemption, so the yield is lower than the coupon rate. 📖 Section 6

**Q12.** For a bond bought at a discount, how does the approximate YTM formula compare with the exact (IRR) YTM in the class examples?
- A) Approximate YTM is far higher (e.g. 15% vs 10%)
- B) Approximate YTM cannot be used for discount bonds
- C) Approximate YTM is always exactly equal to the exact YTM
- D) Approximate YTM is slightly lower (e.g. 10.10% vs ≈ 10.15%)

> ✅ **Answer: D.** The approximation is close but not exact: 10.10% vs ≈ 10.15% for the Infosys-style bond, and 8.89% vs ≈ 8.97% for the L&T-style bond. 📖 Section 6

**Q13.** Why is debt generally considered a cheaper source of finance than equity for a profitable company?
- A) Debt never has to be repaid
- B) Interest is tax-deductible, creating a tax shield
- C) Bondholders get voting rights
- D) Interest is optional, unlike dividends

> ✅ **Answer: B.** Interest reduces taxable income, so the effective cost of debt is lower. Option D is backwards: interest is a contractual obligation, while dividends are discretionary. 📖 Section 1

**Q14.** A company is being liquidated. Which statement is correct?
- A) Bondholders and shareholders are paid equally
- B) Bondholders are paid only if the company made a profit
- C) Bondholders are paid before shareholders
- D) Shareholders are paid before bondholders because they own the company

> ✅ **Answer: C.** Debt has **priority over equity**, including on liquidation. Bondholders are creditors with a contractual claim. 📖 Section 1

**Q15.** A mature company pays a constant dividend of ₹12 per share forever. If the required return is 15%, what is the share price?
- A) ₹180
- B) ₹100
- C) ₹13.80
- D) ₹80

> ✅ **Answer: D.** Zero growth model: P₀ = D₁ / r = 12 / 0.15 = **₹80**. 📖 Section 7A

**Q16.** A company just paid a dividend of ₹10. Dividends grow at 5% forever and the required return is 12%. What is the share price?
- A) ₹87.50
- B) ₹150.00
- C) ₹142.86
- D) ₹210.00

> ✅ **Answer: B.** D₁ = 10 × 1.05 = 10.50, so P₀ = 10.50 / (0.12 − 0.05) = **₹150**. Option C uses D₀ instead of D₁, and Option A ignores growth in the denominator. 📖 Section 7B

**Q17.** An analyst applies the Gordon model to a start-up with r = 10% and g = 12%. What is the correct conclusion?
- A) The share price is negative, so the company should be sold
- B) The share is worth exactly D₁ / 0.10
- C) The model is not valid, because it requires r > g
- D) The share is worth D₁ / 0.02

> ✅ **Answer: C.** With g ≥ r the denominator is zero or negative, so the formula breaks down. Use a two-stage model for a high-growth company instead. 📖 Section 7B

**Q18.** A share trades at ₹200. Next year's expected dividend is ₹10, and dividends are expected to grow at 6% forever. What return are investors requiring?
- A) 11.3%
- B) 5.0%
- C) 6.0%
- D) 11.0%

> ✅ **Answer: D.** Rearranging Gordon: r = D₁/P₀ + g = 10/200 + 0.06 = **11%**. Option A wrongly grows the dividend again (10.60/200 + 6%), even though ₹10 is already D₁. 📖 Section 7B

**Q19.** D₀ = ₹10. Dividends grow at 10% for 2 years and then at 5% forever. The required return is 12%. What is the share price today (two-stage model)?
- A) ₹200.97
- B) ₹164.16
- C) ₹181.50
- D) ₹157.27

> ✅ **Answer: B.** D₁ = 11, D₂ = 12.10; PV = 9.82 + 9.65 = 19.47. D₃ = 12.705, so P₂ = 12.705 / 0.07 = 181.50 and PV = 181.50 / 1.12² = 144.69. P₀ = 19.47 + 144.69 = **₹164.16**. Option A forgets to discount the terminal price, Option C is the undiscounted terminal price, and Option D uses D₂ instead of D₃ for the terminal price. 📖 Section 7C

**Q20.** In a two-stage model with high growth for n years, which dividend is used to calculate the terminal price Pₙ?
- A) Dₙ₊₁ = Dₙ × (1 + g₂)
- B) D₀
- C) Dₙ × (1 + g₁)
- D) Dₙ

> ✅ **Answer: A.** Pₙ = Dₙ₊₁ / (r − g₂): the price at the end of year n is based on the **next** year's dividend, grown at the **stable** rate g₂. 📖 Section 7C

**Q21.** Which statement best contrasts bonds and stocks?
- A) Both have a fixed redemption value at maturity
- B) Stocks are valued using YTM; bonds are valued using the Gordon model
- C) Bonds have contractual cash flows and a redemption value; stocks have uncertain dividends and no redemption value
- D) Stocks have contractual dividends; bonds have uncertain coupons

> ✅ **Answer: C.** Bonds promise fixed coupons plus redemption. Stocks offer growth-linked, uncertain dividends with no redemption, so they are valued with the DDM. 📖 Section 8
