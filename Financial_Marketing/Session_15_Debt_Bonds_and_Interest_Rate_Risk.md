# Session 15: Debt & Bonds – Types, Government Bonds and Interest Rate Risk

> 🎓 In Session 10 we learned to price a bond and estimate its YTM. Today we go one step further: **what kinds of bonds exist, how real government bonds (India vs US) look, and why bond prices move when interest rates change.** The last part, interest rate risk, is the one every bond investor and treasury manager worries about.

**By the end of this session you will be able to:**
- List the core features of debt and the defining characteristics of a bond
- Classify bonds by issuer, security, rate structure and special features
- Tell whether a bond trades at par, premium or discount
- Read a real G-Sec and US T-Note and compare their yields
- Measure interest rate risk using price changes, Macaulay duration and modified duration
- Explain convexity and why duration is only an approximation

---

## 1. Features of Debt

| Feature | Meaning |
|---|---|
| **Obligation to repay** | The borrower is legally bound to return the principal |
| **Interest payment** | The lender is paid interest at a fixed or floating rate |
| **Fixed tenure** | There is a set maturity date after which the loan is fully repaid |
| **Priority over equity** | Debt holders are paid before shareholders, even in liquidation |
| **No ownership rights** | Lenders get no equity stake and no voting rights |
| **Collateral / security** | Debt may be secured by assets or issued unsecured |

---

## 2. What is a Bond?

A **bond** is a fixed-income security that represents a loan made by an investor (the **bondholder**) to a borrower (the **issuer**), usually a government, municipality or corporation.

```
INVESTOR (Bondholder) ──lends principal──▶ ISSUER (Govt / Corporation)
INVESTOR ◀──coupons + principal at maturity── ISSUER
```

Buying a bond = lending money to the issuer in exchange for periodic interest and the return of principal on a set future date.

### Characteristics of a Bond

| Term | Meaning |
|---|---|
| **Face / Par value** | Amount repaid at maturity; the base for calculating the coupon |
| **Coupon rate** | Fixed interest rate paid periodically, as a % of face value |
| **Maturity date** | Date the principal is repaid and the bond ends |
| **Market price** | What the bond trades at today: par, premium or discount, depending on yields |
| **Yield to Maturity (YTM)** | Effective annualised return if you hold the bond until maturity |
| **Credit rating** | An agency's grade of the issuer's ability to repay (risk grade) |
| **Bond type** | Secured/unsecured, fixed/floating, convertible, callable, zero-coupon |
| **Indenture** | The legal contract setting out issuer and bondholder obligations |

---

## 3. Types of Bonds

### 3A. By Issuer

| Issuer | Description | Risk / Yield |
|---|---|---|
| **Government (Sovereign)** | Central government bonds, e.g. Indian G-Secs, US Treasuries | Virtually zero default risk → **lowest yields** |
| **Municipal** | Local governments, for infrastructure; may carry tax benefits in some countries | Low–moderate |
| **Corporate** | Companies raising capital; rated by agencies | **Higher risk, higher yield** |
| **PSU / Public sector** | Government-owned enterprises | Moderate risk, often implicitly backed by the government |

### 3B. By Structure & Features

| Basis | Type | Meaning |
|---|---|---|
| **Security** | Secured | Backed by collateral / specific assets |
| | Unsecured (**Debentures**) | No collateral; relies only on the issuer's creditworthiness |
| **Rate structure** | Fixed-rate | Coupon stays the same throughout |
| | Floating-rate | Coupon resets with a benchmark rate |
| | Zero-coupon | No periodic interest; issued at a deep discount |
| **Special features** | Convertible | Can be converted into equity shares |
| | Callable / Puttable | Issuer (callable) or holder (puttable) can redeem early |
| | Inflation-indexed | Principal adjusts with inflation (CPI) |
| | Perpetual | No maturity date; pays interest indefinitely |

> 💡 **In simple words:** Callable = company ka option (rates gire to company purana mehenga bond wapas le legi). Puttable = investor ka option (rates badhe to investor paisa wapas maang lega). Floating-rate bond ka coupon rates ke saath badalta hai, isliye uska price zyada nahi hilta.

---

## 4. Bond Pricing and Par / Premium / Discount

A bond's price is the **present value of all future coupons plus the face value**, discounted at the required yield:

$$P = \sum_{t=1}^{n} \frac{C}{(1+r)^t} + \frac{F}{(1+r)^n}$$

| Symbol | Meaning |
|---|---|
| P | Price of the bond today |
| C | Periodic coupon = Face × Coupon rate (÷ 2 if semi-annual) |
| r | Required return / discount rate **per period** |
| n | Number of periods until maturity |
| F | Face value, repaid at maturity |

| Case | Condition | Price |
|---|---|---|
| **At par** | Coupon rate = Market yield | Price = Face value |
| **At premium** | Coupon rate > Market yield | Price > Face value |
| **At discount** | Coupon rate < Market yield | Price < Face value |

**Slide example:** A ₹1,000, 8% coupon bond priced to yield 10% trades **below** face value, so it is a discount bond.
Check (10 years, annual coupons): P = 80 × PVIFA(10%, 10) + 1,000 ÷ 1.10¹⁰ = **₹877.11** (≈ ₹875.38 with semi-annual coupons). ✅ Below ₹1,000.

> 💡 **In simple words:** Agar market mein naye bonds 10% de rahe hain aur tumhara bond sirf 8% deta hai, to koi bhi use ₹1,000 mein nahi kharidega. Price girega, jab tak buyer ko total 10% return na mil jaaye.

---

## 5. Real Examples: India vs US Government Bonds

### Example 1: Indian 10-Year G-Sec

| Term | Value |
|---|---|
| Face value | ₹100 |
| Coupon rate | 7% p.a. (₹3.50 every 6 months) |
| Market price | ₹98 (discount) |
| Redemption | ₹100 after 10 years |
| YTM (Sep 2026) | 7.02% (slide) |

You lend ₹98 today, receive ₹3.50 every 6 months, and get ₹100 back in year 10. **YTM > coupon** because you also earn a capital gain of ₹2 (buying at ₹98, receiving ₹100).

> ⚠️ **Slide inconsistency:** at a price of **₹98**, a 10-year 7% semi-annual bond has a YTM of about **7.29%** (approximate formula: (7 + 0.2) ÷ 99 = 7.27%), not 7.02%. A 7.02% YTM matches a price of about **₹99.86**, which is the "before" price the slide uses on the interest-rate-risk chart. Either figure is fine for concepts, but don't mix them in one calculation.

### Example 2: US 10-Year Treasury Note

| Term | Value |
|---|---|
| Face value | \$1,000 |
| Coupon rate | 4.5% p.a. (\$22.50 every 6 months) |
| Market price | \$985 (discount) |
| Redemption | \$1,000 after 10 years |
| YTM (Sep 2026) | 4.8%–5% (slide) |

Rising market yields push existing lower-coupon bonds to trade at a discount.

> ⚠️ **Slide inconsistency:** at **\$985** the YTM is about **4.69%**. A 4.8% YTM gives a price of about **\$976.39**, which is what the interest-rate-risk slide uses (\$976 → \$902).

### Side by Side

| Term | Indian G-Sec | US T-Note |
|---|---|---|
| Face value | ₹100 | \$1,000 |
| Coupon rate | 7% | 4.5% |
| Redemption | ₹100 @ Year 10 | \$1,000 @ Year 10 |
| YTM | 7.02% | 4.8–5% |
| Currency | Indian Rupee | US Dollar |
| Issuer risk | Sovereign (India) | Sovereign (US) |

**Why does India yield more?** Higher **inflation expectations** and a **country risk premium**, compared with the US, which issues the world's reserve currency.

---

## 6. Interest Rate Risk

**Bond prices move inversely to yields.** When market rates rise, existing bonds with fixed coupons become less attractive, so their prices fall.

### The +1% Yield Shock (both 10-year bonds)

| Bond | Yield before → after | Price before | Price after | % change |
|---|---|---|---|---|
| India G-Sec (7% coupon) | 7.02% → 8.02% | ₹99.86 | ₹93.08 | **−6.79%** |
| US T-Note (4.5% coupon) | 4.8% → 5.8% | \$976.39 | \$902.40 | **−7.58%** |

✅ Checked: 93.08 ÷ 99.86 − 1 = −6.79%; 902.40 ÷ 976.39 − 1 = −7.58%.

**The US T-Note falls more.** Its lower coupon means more of its value is "back-loaded" to the final maturity payment, so it is more rate-sensitive.

### Factors that increase interest rate risk

| Factor | Effect on rate sensitivity |
|---|---|
| **Lower coupon** | ⬆️ More sensitive (less cash comes back early) |
| **Longer maturity** | ⬆️ More sensitive (cash flows are further away) |
| **Zero-coupon** | ⬆️ Most sensitive for a given maturity |
| **Floating-rate coupon** | ⬇️ Least sensitive (coupon resets with rates) |

---

## 7. Duration: Measuring Rate Sensitivity

### 7A. Macaulay Duration
The **weighted-average time** (in years) until you receive the bond's cash flows, where each weight is that cash flow's share of the bond's price:

$$D_{Mac} = \frac{\sum t \times PV(CF_t)}{P}$$

### 7B. Modified Duration
Converts Macaulay duration into a **% price change for a 1% change in yield**:

$$D_{Mod} = \frac{D_{Mac}}{1 + y/m} \qquad \%\Delta P \approx -D_{Mod} \times \Delta y$$

(y = annual yield, m = coupon payments per year.)

| Bond | Macaulay duration | Modified duration |
|---|---|---|
| India G-Sec (7%, YTM 7.02%) | 7.35 yrs | 7.10 |
| US T-Note (4.5%, YTM 4.8%) | 8.13 yrs | 7.94 |

✅ Checked: 7.35 ÷ 1.0351 = 7.10; 8.13 ÷ 1.024 = 7.94.

**Why is US duration higher?** Lower coupon → less cash returned early → more value back-loaded → higher duration → more price sensitivity.

| Rule | |
|---|---|
| Higher coupon | → lower duration → less price sensitivity |
| Longer maturity | → higher duration |
| Zero-coupon bond | Duration = maturity (only one cash flow) |

> 💡 **In simple words:** Duration ek "average waiting time" hai ki tumhara paisa kab tak wapas aayega. Jitna zyada wait, utna zyada risk ki rates badalne se price hile. Modified duration seedha batata hai: rate 1% badla to price lagbhag kitne % hilega.

### 7C. Worked Example (extra, not on slides)
A 3-year, ₹1,000 bond with a 6% annual coupon, YTM 7%.

| Year | Cash flow | PV at 7% | t × PV |
|---|---|---|---|
| 1 | 60 | 56.07 | 56.07 |
| 2 | 60 | 52.41 | 104.81 |
| 3 | 1,060 | 865.28 | 2,595.83 |
| **Total** | | **973.76** | **2,756.71** |

- Price = **₹973.76** (discount, since 6% < 7%)
- Macaulay duration = 2,756.71 ÷ 973.76 = **2.83 years**
- Modified duration = 2.83 ÷ 1.07 = **2.65** → a 1% rise in yield cuts the price by about 2.65%.

---

## 8. Convexity: Why the Price-Yield Curve Isn't a Straight Line

Prices fall as yields rise, but the relationship **curves** (it is convex). So duration, which assumes a straight line, is only an **approximation**.

| Yield | India G-Sec (7%) price | US T-Note (4.5%) price, per 100 |
|---|---|---|
| 3% | ~134.3 | ~112.9 |
| 7% | 100.0 (par) | ~82.2 |
| 11% | ~76.1 | ~61.2 |

**Duration estimate vs actual (+1% yield):**

| Bond | Duration estimate | Actual change |
|---|---|---|
| India G-Sec | −7.10% | −6.79% |
| US T-Note | −7.94% | −7.58% |

The actual fall is **smaller** than duration predicts. When yields fall, the actual rise is **larger** than duration predicts. That asymmetry is convexity, and it works in the bondholder's favour.

> 💡 **In simple words:** Duration ek seedhi line maan ke andaaza lagata hai, par asli curve thoda mudi hui hai. Isliye rate badhne par loss andaaze se thoda kam hota hai, aur rate girne par gain andaaze se thoda zyada.

---

## 9. Quick Revision 🧠

- Debt = repayment obligation + interest + fixed tenure + priority over equity + no ownership
- Bond price = PV of coupons + PV of face value
- Coupon = yield → par · coupon > yield → premium · coupon < yield → discount
- Price below face → YTM above coupon (capital gain is added)
- India G-Sec yields more than the US T-Note: inflation + country risk premium
- **Rates up → prices down**; lower coupon / longer maturity / zero-coupon = more sensitive
- Modified duration = Macaulay ÷ (1 + y/m); %ΔP ≈ −D_mod × Δy
- Convexity: duration overstates losses and understates gains

---

## 10. MCQs (Mid–Advanced Level) 📝

**Q1.** A ₹1,000 bond has an 8% coupon, and similar bonds in the market currently yield 10%. How will this bond trade?
- A) At a premium, above ₹1,000
- B) At a discount, below ₹1,000
- C) At exactly ₹800
- D) At par, ₹1,000

> ✅ **Answer: B.** Coupon (8%) < market yield (10%), so the price must fall below face value to give buyers a 10% return. The price is about ₹877 for a 10-year annual-coupon bond. Option C wrongly scales the price by the coupon. 📖 Section 4

**Q2.** A 5-year, ₹1,000 bond pays a 10% annual coupon. If the required yield is 8%, what is its price?
- A) ₹1,100.00
- B) ₹1,079.85
- C) ₹1,000.00
- D) ₹927.90

> ✅ **Answer: B.** P = 100 × PVIFA(8%, 5) + 1,000 ÷ 1.08⁵ = 399.27 + 680.58 = **₹1,079.85**. Coupon > yield, so it trades at a premium. Option D is the price at a 12% yield. 📖 Section 4

**Q3.** What is the price of a 5-year zero-coupon bond with a face value of ₹1,000 if the required yield is 8%?
- A) ₹600.00
- B) ₹1,000.00
- C) ₹680.58
- D) ₹925.93

> ✅ **Answer: C.** P = 1,000 ÷ 1.08⁵ = **₹680.58**. Option D discounts for only one year. 📖 Section 3B

**Q4.** An Indian G-Sec with a 7% coupon (face ₹100) trades at ₹98. What can you say about its YTM?
- A) It is exactly 7%
- B) It cannot be compared with the coupon
- C) It is below 7%
- D) It is above 7%

> ✅ **Answer: D.** Buying below face value adds a capital gain on top of the coupon, so YTM > coupon rate. 📖 Section 5

**Q5.** The 7% G-Sec has a face value of ₹100 and pays coupons semi-annually. How much is each coupon payment?
- A) ₹3.50
- B) ₹3.43
- C) ₹7.00
- D) ₹14.00

> ✅ **Answer: A.** Annual coupon = 7% × ₹100 = ₹7, paid in two halves of **₹3.50**. Option B wrongly applies the rate to the ₹98 market price. 📖 Section 5

**Q6.** A bond has a modified duration of 7.10. If its yield rises by 0.5%, what is the approximate price change?
- A) −14.20%
- B) +3.55%
- C) −7.10%
- D) −3.55%

> ✅ **Answer: D.** %ΔP ≈ −D_mod × Δy = −7.10 × 0.5 = **−3.55%**. Option C uses a 1% change, and Option B has the wrong sign (prices fall when yields rise). 📖 Section 7B

**Q7.** A US T-Note has a Macaulay duration of 8.13 years, a YTM of 4.8% and semi-annual coupons. What is its modified duration?
- A) 7.76
- B) 7.94
- C) 8.52
- D) 8.13

> ✅ **Answer: B.** D_mod = 8.13 ÷ (1 + 0.048/2) = 8.13 ÷ 1.024 = **7.94**. Option A divides by (1 + 0.048), which ignores the semi-annual compounding. 📖 Section 7B

**Q8.** Two 10-year government bonds have the same yield. Bond X has a 7% coupon and Bond Y has a 4.5% coupon. If yields rise by 1%, which statement is correct?
- A) Neither falls, since both are sovereign bonds
- B) Both fall by the same %, since the maturity is the same
- C) Bond Y falls more because more of its value comes at maturity
- D) Bond X falls more because it pays more interest

> ✅ **Answer: C.** A lower coupon means less cash returns early, so value is back-loaded, duration is higher and the price is more rate-sensitive. Sovereign status removes credit risk, not interest rate risk, which is why Option A is wrong. 📖 Section 6

**Q9.** For a +1% yield shock, the slide shows the India G-Sec falling 6.79% although its modified duration is 7.10. Why is the actual fall smaller?
- A) Because of a calculation error on the slide
- B) Because the coupon was paid during the shock
- C) Because sovereign bonds are protected by the central bank
- D) Because of convexity: the price-yield curve bends, so duration overstates the loss

> ✅ **Answer: D.** Duration is a straight-line estimate of a convex curve. For a rise in yields the actual fall is smaller; for a fall in yields the actual gain is larger. 📖 Section 8

**Q10.** What is the Macaulay duration of a 6-year zero-coupon bond?
- A) More than 6 years
- B) About 3 years
- C) 6 years
- D) It cannot be calculated without a coupon

> ✅ **Answer: C.** A zero-coupon bond has a single cash flow at maturity, so its weighted-average time to cash flow equals its maturity. 📖 Section 7B

**Q11.** Which bond is generally the **least** exposed to interest rate risk?
- A) A floating-rate bond whose coupon resets to a benchmark every 6 months
- B) A 10-year 4.5% fixed-rate bond
- C) A 20-year zero-coupon bond
- D) A perpetual fixed-coupon bond

> ✅ **Answer: A.** Its coupon resets with market rates, so its price stays close to par. Option C and Option D have very long-dated cash flows, which makes them highly rate-sensitive. 📖 Section 6

**Q12.** A company issues bonds with no collateral, relying only on its creditworthiness. What are these called?
- A) Inflation-indexed bonds
- B) Secured bonds
- C) Debentures
- D) Municipal bonds

> ✅ **Answer: C.** Unsecured bonds are debentures. Option B is backed by specific assets. 📖 Section 3B

**Q13.** Market interest rates fall sharply. Which feature allows the **issuer** to redeem its high-coupon bonds early and refinance more cheaply?
- A) Callable feature
- B) Convertible feature
- C) Puttable feature
- D) Inflation indexation

> ✅ **Answer: A.** A callable bond gives the issuer the right to redeem early. Option C gives that right to the holder instead. 📖 Section 3B

**Q14.** Why does India's 10-year G-Sec (~7.02%) yield more than the US 10-year T-Note (~4.8–5%), even though both are sovereign?
- A) Indian G-Secs are unsecured debentures
- B) US Treasuries pay coupons annually
- C) Indian G-Secs have a higher face value
- D) Higher inflation expectations and a country risk premium in India

> ✅ **Answer: D.** Investors demand compensation for higher expected inflation and country risk, while the US issues the world's reserve currency. 📖 Section 5

**Q15.** A 3-year, ₹1,000 bond with a 9% annual coupon is priced at a required yield of 9%. What is its price?
- A) ₹1,090.00
- B) ₹1,000.00
- C) ₹1,270.00
- D) ₹917.43

> ✅ **Answer: B.** Coupon rate = required yield, so the bond trades at **par**. Option C simply adds up the undiscounted cash flows. 📖 Section 4

**Q16.** A 3-year, ₹1,000 bond pays a 6% annual coupon and is priced at a 7% yield. What are its price and Macaulay duration?
- A) ₹973.76 and 3.00 years
- B) ₹1,026.73 and 2.83 years
- C) ₹973.76 and 2.83 years
- D) ₹1,000.00 and 3.00 years

> ✅ **Answer: C.** Price = 56.07 + 52.41 + 865.28 = **₹973.76**. Σ t × PV = 2,756.71, so D_Mac = 2,756.71 ÷ 973.76 = **2.83 years**. A coupon bond's duration is always less than its maturity, which rules out Option A. 📖 Section 7C

**Q17.** A US T-Note is priced at \$976.39 with a modified duration of 7.94. Using duration alone, what is the estimated price after a +1% yield shock, and how does it compare with the actual price of \$902.40?
- A) \$905.92, slightly above the actual price
- B) \$902.40, exactly equal to the actual price
- C) \$968.64, far above the actual price
- D) \$898.86, slightly below the actual price

> ✅ **Answer: D.** Estimate = 976.39 × (1 − 0.0794) = **\$898.86**. The actual price is higher because convexity cushions the fall. Option C subtracts only 0.794%, not 7.94%. 📖 Section 8

**Q18.** Which statement about bond types by issuer is correct?
- A) PSU bonds carry no risk at all because they are always explicitly guaranteed
- B) Sovereign bonds have virtually zero default risk and the lowest yields
- C) Corporate bonds have the lowest yields because companies are profitable
- D) Municipal bonds are issued by central governments for defence spending

> ✅ **Answer: B.** Central government bonds carry the least default risk, so they offer the lowest yields. Option A overstates it: PSU bonds are moderate risk and only often implicitly backed. 📖 Section 3A

**Q19.** In which situation does a bond trade at a **premium**?
- A) The bond is a zero-coupon bond
- B) Coupon rate is above market yield
- C) Coupon rate equals market yield
- D) Coupon rate is below market yield

> ✅ **Answer: B.** A higher-than-market coupon makes the bond attractive, so buyers pay more than face value. Option A always trades at a discount (before maturity) because it pays no coupon. 📖 Section 4

**Q20.** An inflation-indexed bond protects the investor mainly by:
- A) Adjusting the principal with the Consumer Price Index (CPI)
- B) Having no maturity date
- C) Letting the holder convert into equity shares
- D) Allowing the issuer to redeem early when inflation falls

> ✅ **Answer: A.** The principal (and so the coupon calculated on it) moves with inflation, preserving real value. Option C describes a convertible bond and Option B a perpetual bond. 📖 Section 3B

**Q21.** A perpetual bond pays a coupon of ₹50 a year forever. If the required yield is 8%, what is it worth?
- A) ₹625.00
- B) ₹1,000.00
- C) ₹540.00
- D) ₹400.00

> ✅ **Answer: A.** Perpetuity value = C ÷ r = 50 ÷ 0.08 = **₹625**. Option D multiplies by 8 instead of dividing by 0.08. 📖 Section 3B

**Q22.** Which combination of features gives the **highest** duration?
- A) Low coupon (or zero coupon), long maturity
- B) Low coupon, short maturity
- C) High coupon, long maturity
- D) High coupon, short maturity

> ✅ **Answer: A.** Duration rises with maturity and falls with coupon, so a low/zero coupon with long maturity pushes the most value to the far future. 📖 Section 7B
