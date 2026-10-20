# Session 9: Cost of Capital (Cost of Equity, Risk, Beta & Alpha)

> 🎓 Sessions 1–8 covered statements, ratios and management accounting. Now we ask a finance question: **what return must a company give its investors?** That required return is the **cost of capital**. Today we focus on the **equity** side: how risky a share is (beta) and what return shareholders demand (CAPM).

**By the end of this session you will be able to:**
- Explain cost of capital and the features of equity shares (incl. dual-class shares)
- Separate **systematic** from **unsystematic** risk
- Calculate **cost of equity** using **CAPM**
- Estimate **historical beta** from return data
- Calculate and interpret **alpha**
- Compare CAPM with the Dividend Growth Model

> ⚠️ The learning-outcomes slide also lists after-tax cost of debt, WACC and capital-structure effects, but this deck does not cover them. Cost of debt comes in Session 10.

---

## 1. Cost of Capital: the Big Idea

A company raises money from two main sources: **equity** (shareholders) and **debt** (lenders). Each source expects a return.

| Term | Meaning |
|---|---|
| **Cost of Equity (Ke)** | Minimum return shareholders require |
| **Cost of Debt (Kd)** | Return lenders require (interest) |
| **WACC** | Weighted average of Ke and Kd, using the mix of equity and debt as weights |

> 💡 **In simple words:** Investor ne paisa diya hai, to wo return expect karega. Company ke liye wahi expected return uski "cost" hai. Agar project is cost se kam kamata hai, to wo investors ka paisa barbaad kar raha hai.

---

## 2. Features of Equity Shares

| # | Feature | Meaning |
|---|---|---|
| 1 | **Residual claim** | Paid **last** (after debt and preference holders), but owns all remaining profits/assets |
| 2 | **Voting rights** | Usually one share = one vote on major decisions |
| 3 | **Dividends** | **Not fixed or guaranteed**; paid at the board's discretion |
| 4 | **Perpetual capital** | No maturity date; stays on the balance sheet indefinitely |
| 5 | **Limited liability** | Maximum loss = amount invested |
| 6 | **Capital appreciation** | Returns come mainly from share-price growth, not fixed interest |

> 📌 Because equity holders are paid last and get no guaranteed dividend, they carry more risk than lenders. So **Ke is normally higher than Kd**.

### 2A. Ordinary Shares vs Dual-Class Shares

| Ordinary shares | Dual / multi-class shares |
|---|---|
| One share = one vote | **Same economic (dividend) rights, different voting rights** |
| Voting power matches investment | Founders hold **high-vote** shares to keep control |
| Standard for most listed companies | Public gets low- or no-vote shares |
| No unequal classes | Used by Alphabet, Meta and other tech firms to raise capital **without losing control** |

**Case: Alphabet (three classes)**

| Class | Votes per share | Traded? |
|---|---|---|
| Class A (GOOGL) | 1 | Yes |
| Class B | 10 | No (held only by founders) |
| Class C (GOOG) | 0 | Yes |

Larry Page and Sergey Brin hold **~52.7% of voting power with under 12% of the economic stake**.

**Case: Meta (two classes)**

| Class | Votes per share | Holder |
|---|---|---|
| Class A (META) | 1 | Public |
| Class B | 10 | Almost entirely Mark Zuckerberg (99.7% of Class B) |

This gives Zuckerberg **~61% of voting power with only ~13–14% of the economic interest**.

> 🎯 A small economic stake can give majority control. This is the classic **separation of ownership and control**.

---

## 3. Systematic vs Unsystematic Risk ⭐

### 3A. Systematic Risk (Market risk, Non-diversifiable)
Affects the **whole market/economy**. It comes from macroeconomic, political or social forces that no single company controls. **It cannot be diversified away.**

| Example | Why it hits everyone |
|---|---|
| Interest rate hikes | Costlier borrowing pushes valuations down across the market |
| Inflation | Erodes purchasing power and margins in all sectors |
| Recession (2008 crisis) | Markets crashed worldwide: banks, retailers, tech |
| Pandemic (COVID-19) | Disrupted supply, demand and labour in every industry |

### 3B. Unsystematic Risk (Specific risk, Diversifiable)
Affects **one company or industry only**. It can be reduced or removed by holding a diversified portfolio.

| Example | Who is hit |
|---|---|
| Failed acquisition | Only that firm's stock |
| Labour strike | One manufacturer's plant |
| Product recall | One pharma company, not the sector |
| Fraud / accounting scandal | That specific stock |

### 3C. Comparison

| Basis | Systematic Risk | Unsystematic Risk |
|---|---|---|
| Scope | Entire market/economy | Specific company/industry |
| Source | Macroeconomic, political, social | Firm-specific (management, labour) |
| Diversifiable? | **No** | **Yes** |
| Also called | Market risk | Specific / residual risk |
| Measured by | **Beta (β)** | Std. deviation of firm-specific returns |
| Priced by CAPM? | **Yes**, investors are compensated | **No**, removed by diversification at no cost |

> 💡 **In simple words:** 30–40 alag-alag sectors ke shares le lo, to ek company ka scandal ya strike ka asar almost khatam ho jayega (unsystematic). Par RBI rate badhaye ya recession aaye, to sab gir jayenge (systematic). Market sirf usi risk ka extra return deta hai jo tum hata nahi sakte.

---

## 4. CAPM: Cost of Equity ⭐

$$K_e = R_f + \beta \times (R_m - R_f)$$

| Symbol | Name | Meaning |
|---|---|---|
| **Ke** | Cost of equity | Minimum return equity investors require |
| **Rf** | Risk-free rate | Return on a riskless asset (e.g. 10-year G-Sec) |
| **β** | Beta | Sensitivity of the stock to market movements |
| **Rm** | Expected market return | Expected return on the broad market index |
| **(Rm − Rf)** | Market risk premium | Extra return demanded for bearing market risk |

> 💡 **In simple words:** Ke = "bina risk ka return" + "risk ka bonus". Bonus = market ka extra return × stock kitna risky hai (beta).

### 4A. Understanding Beta

| Beta | Meaning | Typical examples |
|---|---|---|
| **β = 1** | Moves in line with the market | Index-like stocks |
| **β > 1** | More volatile than the market (aggressive) | Tech, cyclicals |
| **β < 1** | Less volatile (defensive) | FMCG, utilities |
| **β = 0** | No correlation with the market | Theoretical, risk-free-like |
| **β < 0** | Moves opposite to the market (rare) | Gold-type assets |

> ⚠️ The slide gives "gold miners" as the β < 0 example. Gold itself often moves against the market, but gold-mining **shares** usually have a positive beta. Treat it as a loose example.

📌 Rule of thumb: **higher beta → higher Ke**; lower beta → lower Ke.

### 4B. CAPM Examples (Rf = 7.0%, Rm = 13%)

| Company | β | Calculation | Ke | Reading |
|---|---|---|---|---|
| Reliance Industries | 0.83 | 7% + 0.83 × (13% − 7%) = 7% + 4.98% | **11.98%** | Less volatile than the Sensex |
| TCS | 1.13 | 7% + 1.13 × 6% = 7% + 6.78% | **13.78%** | Slightly more volatile, so higher required return |
| Infosys (Rm = 12%) | 1.1 | 7% + 1.1 × (12% − 7%) = 7% + 5.5% | **12.5%** | |

✔️ Check: TCS − Reliance = 13.78 − 11.98 = 1.80% = (1.13 − 0.83) × 6% ✅

> ⚠️ The CAPM example image writes "Rf = 7% + 1.1 × (12% − 7%)"; the left side should be **Re/Ke**, not Rf. The same image also shows Reliance β = 1.20, while slide 22 uses 0.83 and the historical calculation gives ~1.65. Beta depends on period, frequency and data (see Section 5C), so don't mix these numbers.

---

## 5. Estimating Historical Beta

### 5A. Two methods

**1. Covariance / Variance method**
$$\beta = \frac{Cov(R_i, R_m)}{Var(R_m)}$$
- Cov(Ri, Rm): how the stock's returns move with the market's returns
- Var(Rm): the market's own volatility

**2. Regression (slope) method**
$$R_i = \alpha + \beta \times R_m + \varepsilon$$
- **α (alpha):** stock return unrelated to the market
- **β (beta):** the slope = systematic risk
- **ε (epsilon):** error term = firm-specific (unsystematic) risk

Shortcut form of the regression slope:
$$\beta = \frac{n\Sigma(R_iR_m) - \Sigma R_i \times \Sigma R_m}{n\Sigma R_m^2 - (\Sigma R_m)^2}$$

**Steps:** (1) find the means of Ri and Rm → (2) deviations from each mean → (3) multiply deviations and sum → covariance → (4) square the Rm deviations and sum → variance → (5) β = Cov / Var.

### 5B. Solved Example: Reliance vs BSE Sensex (5 hypothetical months)

**Step 1–3: Returns and averages**
- Avg Reliance = (4 + 2 − 3 + 5 − 2) ÷ 5 = **1.2%**
- Avg Sensex = (2 + 1 − 1 + 3 − 2) ÷ 5 = **0.6%**

**Step 4–5: Deviations, cross-products, squares**

| Month | Ri | Rm | Ri − R̄i | Rm − R̄m | Cross-product | (Rm − R̄m)² |
|---|---|---|---|---|---|---|
| Jan | 4% | 2% | 2.8% | 1.4% | 0.000392 | 0.000196 |
| Feb | 2% | 1% | 0.8% | 0.4% | 0.000032 | 0.000016 |
| Mar | −3% | −1% | −4.2% | −1.6% | 0.000672 | 0.000256 |
| Apr | 5% | 3% | 3.8% | 2.4% | 0.000912 | 0.000576 |
| May | −2% | −2% | −3.2% | −2.6% | 0.000832 | 0.000676 |
| **Total** | | | | | **0.002840** | **0.001720** |

**Step 6: Covariance and variance** (sample data, divide by n − 1 = 4)
- Cov = 0.002840 ÷ 4 = **0.000710**
- Var = 0.001720 ÷ 4 = **0.000430**

**Step 7: Beta**
$$\beta = 0.000710 \div 0.000430 \approx \mathbf{1.65}$$

✔️ Check: the (n − 1) cancels, so β = 0.002840 ÷ 0.001720 = 1.651 ✅

**Reading:** a 1% market move is linked, on average, to a ~1.65% move in Reliance in the same direction. This is a **historical estimate**, not a prediction.

> 💡 **In simple words:** Beta = "market 1% hila to stock kitna hila". Covariance batata hai dono saath-saath kitna chalte hain; variance se divide karke hum ise market ki apni movement ke hisaab se scale kar dete hain.

### 5C. Practice Example (slide 23, answer worked here)

| Month | Ri (%) | Rm (%) |
|---|---|---|
| 1 | 2.5 | 1.8 |
| 2 | −1.2 | −0.9 |
| 3 | 3.0 | 2.2 |
| 4 | −0.5 | −0.3 |
| 5 | 1.8 | 1.5 |

- ΣRi = 5.6, ΣRm = 4.3, Σ(RiRm) = 15.03, ΣRm² = 11.23, n = 5
- Numerator = 5 × 15.03 − 5.6 × 4.3 = 75.15 − 24.08 = 51.07
- Denominator = 5 × 11.23 − 4.3² = 56.15 − 18.49 = 37.66
- **β = 51.07 ÷ 37.66 ≈ 1.36**

✔️ Check with deviations: Σ cross-products = 10.214, Σ squared Rm deviations = 7.532 → 10.214 ÷ 7.532 = 1.356 ✅

> 📌 Beta changes with the **time period, return frequency and benchmark index**. That is why different data providers report different betas for the same stock.

---

## 6. Alpha (α): Skill vs Risk

Alpha is the **excess return** over what CAPM predicts for the risk taken. It separates skill from risk-taking.

$$\alpha = R_{actual} - [R_f + \beta(R_m - R_f)]$$

| Alpha | Meaning |
|---|---|
| α = 0 | Performed exactly as expected for its risk |
| α > 0 | **Outperformed** its risk-adjusted expectation |
| α < 0 | **Underperformed** its risk-adjusted expectation |

### Example 1: Risk-adjusted comparison (Rf = 4%, Benchmark Rm = 10%)

| | Fund A | Fund B |
|---|---|---|
| Beta | 1.5 | 0.8 |
| Actual return | 14% | 12% |
| CAPM expected | 4 + 1.5 × 6 = **13%** | 4 + 0.8 × 6 = **8.8%** |
| **Alpha** | **+1.0%** | **+3.2%** |

Fund B had the lower headline return but took less risk and added **more excess return**, so it shows more manager skill.

### Example 2: Active vs passive fund
Two friends invest ₹1 lakh at similar risk: Nifty 50 index fund = 20%, active large-cap fund = 25%. **Alpha ≈ +5%**, which reflects the manager's stock selection and timing skill.

> ⚠️ Example 2 simply subtracts the index return. That equals CAPM alpha only if the active fund's beta ≈ 1 ("similar risk"). With a different beta, use the full formula.

> 💡 **In simple words:** Zyada return dekh ke impress mat ho. Pehle pucho "kitna risk liya?" Alpha batata hai risk ke hisaab se jitna milna chahiye tha, usse kitna extra mila.

---

## 7. CAPM vs Dividend Growth Model (DGM)

The DGM estimates Ke from dividends (standard form, not written on the slide): **Ke = D1 / P0 + g**, where D1 = next year's dividend, P0 = current price, g = constant growth rate.

| Basis | CAPM | Dividend Growth Model |
|---|---|---|
| Applicability | All listed companies (dividend or not) | **Only dividend-paying** companies |
| Risk measurement | Explicit, via beta | Implicit / ignored |
| Data required | Rf, β, Rm | Price, expected dividend, growth rate |
| Key assumption | Efficient markets | Constant, perpetual dividend growth |
| Sensitivity | Beta and market-premium estimates | **Highly sensitive to g** |
| Best suited for | Growth/tech and all-sector firms | Stable, mature, dividend-paying firms |

> 📌 The two models are **complementary**, not competing. Analysts often compute both and compare.

---

## 8. Quick Revision 🧠

- **Cost of capital** = return investors require; WACC blends Ke and Kd by their weights
- Equity: residual claim, voting, no fixed dividend, perpetual, limited liability → riskier than debt → **Ke > Kd**
- **Dual-class shares:** same economic rights, different votes → founders keep control
- **Systematic risk** (market, non-diversifiable, measured by β, priced by CAPM) vs **Unsystematic** (firm-specific, diversifiable, not priced)
- **CAPM:** `Ke = Rf + β(Rm − Rf)`
- **Beta:** `Cov(Ri, Rm) / Var(Rm)`; β > 1 aggressive, β < 1 defensive
- **Alpha:** `Actual − CAPM expected`; positive = skill
- **DGM:** `Ke = D1/P0 + g`; only for dividend payers, very sensitive to g

---

## 9. MCQs (Mid–Advanced Level) 📝

**Q1.** The risk-free rate is 6%, a stock's beta is 1.4 and the expected market return is 11%. What is the cost of equity under CAPM?
- A) 15.4%
- B) 13.0%
- C) 7.0%
- D) 21.4%

> ✅ **Answer: B.** Ke = 6% + 1.4 × (11% − 6%) = 6% + 7% = 13%. Option A multiplies beta by the full market return instead of the risk premium, and Option C forgets to add Rf. 📖 Section 4

**Q2.** A stock has Ke = 13.6% and β = 1.2 when Rf = 7%. What expected market return does this imply?
- A) 14.9%
- B) 5.5%
- C) 13.6%
- D) 12.5%

> ✅ **Answer: D.** 13.6 = 7 + 1.2 × (Rm − 7), so Rm − 7 = 6.6 ÷ 1.2 = 5.5, and Rm = 12.5%. Option B is the market risk premium, not Rm. 📖 Section 4

**Q3.** Which type of risk does CAPM compensate investors for?
- A) Unsystematic risk
- B) Total risk (systematic + unsystematic)
- C) Systematic risk
- D) Only firm-specific risk

> ✅ **Answer: C.** Unsystematic risk can be diversified away at no cost, so the market does not pay a premium for it. Only systematic risk, measured by beta, is priced. 📖 Section 3C

**Q4.** Which of the following is an example of **systematic** risk?
- A) The central bank raising interest rates sharply
- B) A product recall at one pharmaceutical company
- C) An accounting fraud at a listed company
- D) A labour strike at one car plant

> ✅ **Answer: A.** An interest-rate hike affects borrowing costs and valuations across the whole market. The other events are firm-specific. 📖 Section 3A

**Q5.** An investor holds 40 stocks spread across many sectors. Which risk mainly remains in the portfolio?
- A) Labour strike risk
- B) Product recall risk
- C) Management failure risk
- D) Market (systematic) risk

> ✅ **Answer: D.** Diversification removes most firm-specific risk, but recessions, inflation and rate changes still hit every stock. 📖 Section 3

**Q6.** A stock has β = 0.7. If the market rises by 10%, what is the expected move in the stock, based on beta alone?
- A) Falls about 7%
- B) Rises about 14.3%
- C) Rises about 10%
- D) Rises about 7%

> ✅ **Answer: D.** Expected move ≈ β × market move = 0.7 × 10% = 7%, in the same direction. Option B divides by beta instead of multiplying. 📖 Section 4A

**Q7.** Cov(Ri, Rm) = 0.00048 and Var(Rm) = 0.00040. What is the beta?
- A) 0.83
- B) 1.20
- C) 0.00008
- D) 1.00

> ✅ **Answer: B.** β = 0.00048 ÷ 0.00040 = 1.2. Option A inverts the ratio (Var ÷ Cov), and Option C subtracts instead of dividing. 📖 Section 5A

**Q8.** While estimating beta from 6 months of data, an analyst divides both covariance and variance by n instead of n − 1. What happens to beta?
- A) It stays the same
- B) It becomes negative
- C) It falls by 20%
- D) It rises by 20%

> ✅ **Answer: A.** The same divisor appears in the numerator and denominator, so it cancels. β = Σ cross-products ÷ Σ squared market deviations either way. 📖 Section 5B

**Q9.** Sum of cross-products of deviations = 0.0036 and sum of squared market deviations = 0.0024 (n = 6). What is the beta?
- A) 0.0012
- B) 1.50
- C) 7.50
- D) 0.67

> ✅ **Answer: B.** β = 0.0036 ÷ 0.0024 = 1.5 (dividing both by n − 1 = 5 gives the same result). Option D inverts the ratio. 📖 Section 5B

**Q10.** In the regression Ri = α + βRm + ε, what does ε represent?
- A) The market risk premium
- B) Systematic risk
- C) Firm-specific (unsystematic) risk
- D) The risk-free rate

> ✅ **Answer: C.** The error term captures stock movements not explained by the market, which is firm-specific risk. Beta (the slope) captures systematic risk. 📖 Section 5A

**Q11.** Rf = 5%, Rm = 11%, a fund's β = 1.2 and its actual return is 14%. What is its alpha?
- A) −1.8%
- B) +1.8%
- C) +3.0%
- D) +9.0%

> ✅ **Answer: B.** Expected = 5 + 1.2 × 6 = 12.2%. Alpha = 14 − 12.2 = +1.8%. Option C compares with Rm instead of the CAPM expected return. 📖 Section 6

**Q12.** Rf = 4% and Rm = 10%. Fund X (β = 1.5) returned 15%; Fund Y (β = 0.6) returned 10%. Which statement is correct?
- A) Fund X has the higher alpha because its return is higher
- B) Fund Y has a negative alpha
- C) Fund Y has the higher alpha (+2.4% vs +2.0%)
- D) Both have the same alpha

> ✅ **Answer: C.** X expected = 4 + 1.5 × 6 = 13%, so α = +2.0%. Y expected = 4 + 0.6 × 6 = 7.6%, so α = +2.4%. A higher headline return does not mean higher skill. 📖 Section 6

**Q13.** A fund has a negative alpha. What does this mean?
- A) Its beta is negative
- B) It took no market risk
- C) Its return was negative
- D) It earned less than CAPM predicts for the risk it took

> ✅ **Answer: D.** Alpha compares actual return with the risk-adjusted expected return. A fund can earn a positive return and still have negative alpha. 📖 Section 6

**Q14.** For which company is the Dividend Growth Model least suitable for estimating cost of equity?
- A) A stable FMCG company with a long dividend history
- B) A growing tech firm that pays no dividends
- C) A mature utility with steady dividends
- D) A large bank with a regular payout policy

> ✅ **Answer: B.** The DGM needs a dividend stream. A non-dividend payer has no D1, so CAPM is the better tool. 📖 Section 7

**Q15.** Which statement about CAPM vs DGM is correct?
- A) DGM results are highly sensitive to the assumed growth rate
- B) CAPM assumes constant perpetual dividend growth
- C) CAPM works only for dividend-paying firms
- D) DGM measures risk explicitly through beta

> ✅ **Answer: A.** In Ke = D1/P0 + g, a small change in g moves Ke directly. The other statements swap the features of the two models. 📖 Section 7

**Q16.** In a dual-class share structure, what usually differs between the share classes?
- A) Dividend rights
- B) Limited liability
- C) Voting rights
- D) Face value only

> ✅ **Answer: C.** Dual-class shares have the same economic (dividend) rights but different voting rights, so founders can keep control. 📖 Section 2A

**Q17.** Which of Alphabet's share classes carries **no** voting rights?
- A) Class C (GOOG)
- B) Class A (GOOGL)
- C) All three carry equal votes
- D) Class B

> ✅ **Answer: A.** Class C (GOOG) is publicly traded with zero votes. Class A has 1 vote, and Class B has 10 votes and is held by the founders. 📖 Section 2A

**Q18.** Mark Zuckerberg has ~61% of Meta's voting power with only ~13–14% of the economic interest. This best illustrates:
- A) Unsystematic risk
- B) Separation of ownership and control
- C) Limited liability
- D) The going concern concept

> ✅ **Answer: B.** A small economic stake gives majority voting control through high-vote Class B shares. 📖 Section 2A

**Q19.** Which of the following is **not** a feature of equity shares?
- A) A fixed, guaranteed dividend
- B) Residual claim on profits and assets
- C) Limited liability
- D) No maturity date

> ✅ **Answer: A.** Equity dividends are paid at the board's discretion and are not guaranteed. A fixed return is a feature of debt. 📖 Section 2

**Q20.** Using Rf = 7% and Rm = 13%, Reliance has β = 0.83 and TCS has β = 1.13. By how much is TCS's cost of equity higher?
- A) 1.13 percentage points
- B) 6.00 percentage points
- C) 0.30 percentage points
- D) 1.80 percentage points

> ✅ **Answer: D.** Difference = (1.13 − 0.83) × 6% = 1.8 points (13.78% − 11.98%). Option C is just the beta difference, without multiplying by the risk premium. 📖 Section 4B

**Q21.** An asset has β = −0.3. If the market falls 10%, what is its expected move, based on beta alone?
- A) Falls 3%
- B) No change
- C) Rises 3%
- D) Falls 13%

> ✅ **Answer: C.** −0.3 × (−10%) = +3%. A negative-beta asset moves against the market, which is why it can act as a hedge. 📖 Section 4A

**Q22.** Two data providers report different betas for the same stock. What is the most likely reason?
- A) They used different time periods, return frequencies or benchmark indices
- B) One of them made an arithmetic mistake
- C) Beta is fixed by the stock exchange
- D) Beta depends on the company's dividend policy

> ✅ **Answer: A.** Beta is a statistical estimate. It changes with the period, the frequency (daily, weekly, monthly) and the benchmark chosen. 📖 Section 5C
