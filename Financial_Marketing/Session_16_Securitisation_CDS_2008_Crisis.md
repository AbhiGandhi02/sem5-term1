# Session 16: Financial Derivatives – Special Products

> 🎓 Securitisation and MBS, credit spreads and credit default swaps (CDS), CDOs and the 2008 crisis, and market-linked notes. The deck's running theme: **every idea maps to reliability engineering.** Watch for **independent vs common-mode failure**.

**After this session you should be able to:**
- Explain how securitisation turns illiquid loans into tradable securities, and how tranches split risk and return
- Run a tranche **cash waterfall** and a **loss waterfall** with numbers
- Calculate a credit spread, a CDS premium, a CDS payout and the market-implied default chance
- Explain how CDOs, leverage and CDS turned US subprime losses into a global crisis in 2008
- Break a market-linked note into a bond + options, compute its payoff, and list its risks

*Recap:* Session 10 priced plain bonds and YTM. Here we look at what happens when loans are pooled, sliced and insured.

---

## 1. Vocabulary (with the deck's engineering analogies)

| Term | Meaning | Engineering analogy |
|---|---|---|
| Principal | The amount lent | The nominal load |
| Interest | The price of borrowing | Rent on capital |
| Default | Borrower fails to repay | A component failure |
| Yield | Return earned; riskier means higher | A risk premium |
| Basis point (bp) | 1/100 of 1%, so **200 bp = 2%** | A fine tolerance unit |
| Derivative | A contract whose value depends on something else | A sensor reading that triggers a payout |

---

## 2. Securitisation and Mortgage-Backed Securities (MBS)

**Securitisation** turns many slow, illiquid loans into securities that investors can buy and sell. When the loans are home loans, the securities are **mortgage-backed securities (MBS)**.

### 2A. How it works (4 steps)

| Step | What happens |
|---|---|
| 1. Banks lend | Banks and housing finance companies lend to thousands of borrowers |
| 2. Loans are pooled | Loans are sold to a separate legal entity, the **SPV (Special Purpose Vehicle)**, off the bank's books |
| 3. Tranches | The securities are cut into **senior, mezzanine and equity** layers by who absorbs losses first |
| 4. Investors get EMIs | Borrowers' monthly payments pass through to investors; the bank recovers cash to lend again |

**Flow:** Borrowers → Lender (originates and usually **services** the loans) → SPV (buys the pool, issues tranches) → Investors.
- **Loans and risk move right:** lender → SPV → investors.
- **Cash moves left, then right again:** investors' money funds the SPV's purchase, the lender gets cash to lend again, and borrowers' EMIs flow back through the servicer to investors.

> 💡 **In simple words:** Bank ne 1,000 home loans diye, unhe ek alag company (SPV) ko bech diya, aur SPV ne un loans ke EMIs ka "haq" investors ko slices mein bech diya. Bank ko paisa turant wapas, investors ko EMIs.

### 2B. Benefits vs risks

| Benefits | Risks |
|---|---|
| **Liquidity:** a 20-year loan becomes a security that can be sold in a day | **Moral hazard:** if the seller keeps nothing, it has less reason to check borrower quality |
| **Capital recycling:** the lender gets cash back and lends again, so the same capital funds many more homes | **Opacity:** buyers cannot inspect thousands of loans, so they rely on ratings and models |
| **Diversification:** one investor holds a slice of thousands of borrowers | **Correlation:** diversification fails when one cause hits every loan |
| **Choice:** cautious and risk-seeking investors each buy the risk level they want | **Complexity:** each extra layer makes the real risk harder to see |

### 2C. Tranches: the cash waterfall ⭐

Illustrative pool: **₹500 crore of loans at 9% earns ₹45 crore of interest a year.** Payments flow down in strict order.

| Order | Tranche | Size | Promised return | Paid a year |
|---|---|---|---|---|
| 1st | Senior | ₹400 cr | 7.5% | ₹30 cr |
| 2nd | Mezzanine | ₹70 cr | 11% | ₹7.7 cr |
| 3rd | Equity | ₹30 cr | Whatever is left | ₹7.3 cr (≈ 24%) |

✔️ Check: 30 + 7.7 + 7.3 = ₹45 cr. Equity return = 7.3 ÷ 30 = 24.3%.

**If borrowers pay less:**
- Pool pays ₹40 cr → senior 30, mezzanine 7.7, **equity only ₹2.3 cr**.
- Pool pays ₹32 cr → senior 30, mezzanine gets 2 (**short by ₹5.7 cr**), equity **nothing**.

*Pipe network view:* the senior pipe fills first, so it is safest and earns least. The equity pipe fills last, so it earns most when all goes well and runs dry first.

### 2D. The loss waterfall (subordination)

**Losses run in reverse:** equity absorbs the first losses, then mezzanine, then senior. This is **subordination**, the main **credit enhancement** in a securitisation. (*Engineering view:* tranching works like crumple zones; equity takes the damage first, senior last.)

**Rule:** Equity absorbs losses up to ₹30 cr → mezzanine up to the next ₹70 cr (total ₹100 cr) → only losses above ₹100 cr hit senior.

| Scenario | Pool loss | Equity (₹30 cr) | Mezzanine (₹70 cr) | Senior (₹400 cr) |
|---|---|---|---|---|
| Normal defaults | ₹25 cr | Loses ₹25 cr | Safe | Safe |
| Bad year | ₹60 cr | Wiped out | Loses ₹30 cr | Safe |
| Nationwide crash | ₹150 cr | Wiped out | Wiped out | Loses ₹50 cr (12.5%) |

### 2E. Why tranching depends on independence ⭐

| Independent defaults | Common-mode failure |
|---|---|
| Each of 1,000 loans has a 3% default chance, unrelated to the others | A nationwide price fall raises every borrower's default chance **at once** |
| Losses cluster near 3% of the pool ≈ ₹15 cr of ₹500 cr | Losses can reach 30% of the pool = ₹150 cr |
| Equity (₹30 cr) absorbs it; senior untouched | Equity and mezzanine wiped out; senior loses ₹50 cr |

> 🎯 **Rule of thumb:** Diversification protects against **independent** failures, not against a **shared cause**.

> 💡 **In simple words:** 1,000 alag-alag logon ka default ek saath hona mushkil hai, jab tak koi ek common wajah (jaise poore desh mein house prices girna) sab ko ek saath na maare. Tab diversification kaam nahi karta.

### 2F. MBS in practice

**Follow one loan:** a Bengaluru buyer borrows ₹60 lakh for 20 years at 9%. EMI ≈ **₹54,000**, so the lender collects ≈ **₹1.3 crore** over two decades. Sold into a pool, the lender gets ₹60 lakh back **now** and lends it again.

| | Example 1: US agency MBS (Ginnie Mae) | Example 2: India's PTCs |
|---|---|---|
| Structure | Bank makes government-insured loans (FHA/VA), pools them, issues securities **guaranteed by Ginnie Mae** (first pass-through 1970). A servicer collects payments, keeps a small fee, passes interest + principal on | A vehicle-loan NBFC sells ₹500 cr of truck loans to a **trust**, which issues **pass-through certificates (PTCs)** to banks, mutual funds, insurers. The NBFC stays the servicer |
| Key point | **Credit risk is removed** by the guarantee, but a new risk appears: **prepayment risk** | Rating agencies rate each tranche; **credit enhancement** (cash reserve or junior tranche) protects seniors |
| Why it matters | Made 30-year mortgages cheap and widely available | Cash returns in months, not years; investors get granular retail loans; **post-2008 RBI rules require originators to retain a share of the risk** ("skin in the game") |

> **Prepayment risk:** when rates fall, homeowners refinance, investors get principal back early and must reinvest at **lower** rates. (Like a tank whose outlet opens wider when conditions change.)

---

## 3. Credit Spreads

**Credit spread = company bond yield − government bond yield.**
Example: 9% − 7% = 2% = **200 bp**. A wider spread means the market fears default more.

**Rule of thumb:**
$$\text{Spread} \approx \text{Chance of default} \times \text{Loss if default} \quad\Rightarrow\quad \text{Implied default chance} = \frac{\text{Spread}}{\text{Loss if default}}$$

Loss if default = 1 − recovery rate (failure rate × consequence, the same logic as risk in safety engineering).

| Observed spread | Loss if default | Implied yearly default chance |
|---|---|---|
| 2% (200 bp) | 60% (40% recovered) | 2 ÷ 0.6 ≈ 3.3% |
| 6% (600 bp) | 60% | 6 ÷ 0.6 = 10% |
| 0.5% (50 bp) | 60% | 0.5 ÷ 0.6 ≈ 0.8% |

- **What moves a spread:** the borrower's earnings and debt, the economy, and investor fear. In a panic, spreads widen **even for sound borrowers**.
- A spread is a **market-implied failure rate that updates daily**: a sensor reading, not a forecast, and it includes a **fear premium**.

---

## 4. Credit Default Swaps (CDS)

A **CDS is like a warranty (insurance) on a loan.**

| Party | Role |
|---|---|
| **Protection buyer** | Pays a fixed premium (spread) each period. Often a bank/fund worried about a borrower, or a trader betting on its decline |
| **Protection seller** | Collects the premium and pays out if a credit event occurs. Often a large insurer or bank |
| **Reference entity** | The company, country or security being insured. **Not a party to the contract** and may not know it exists |

- **Credit events:** failure to pay, bankruptcy, and in many contracts a debt restructuring.
- **Settlement:** the seller either pays **face value − recovered amount** in cash, or takes the defaulted bond and pays face value.
- **Premium per year = Notional × spread.** ₹100 cr at 200 bp = ₹2 cr/year; at 600 bp, the same cover costs ₹6 cr.
- **Two surprises:** a buyer needs **no loss** to collect (a **"naked" CDS**), and the seller's ability to pay is **not guaranteed**, so the insurance chain can break.

### Solved example: CDS step by step

| Step | What happens | Numbers |
|---|---|---|
| 1. Contract | Bank buys protection on a company's bonds | ₹100 cr notional, 200 bp |
| 2. Premium | Bank pays the seller every year | ₹2 cr a year |
| 3. Stress | Company's health worsens; spread widens | 200 bp → 600 bp |
| 4. Default | Bondholders recover only 40% | Loss = ₹60 cr |
| 5. Payout | Seller pays the bank the loss | ₹60 cr |

✔️ Payout = 100 × (1 − 0.40) = ₹60 cr.

**Counterparty risk:** if the seller cannot pay, the protection is worthless. *US version:* a bank holding a \$100M AAA CDO layer buys CDS at 0.5% = \$500,000 a year; if the layer loses 30%, the seller owes \$30M.

### Two real cases

| AIG (2008) | Credit Suisse (March 2023) |
|---|---|
| AIG's financial products unit sold CDS on the top "safest" layers of mortgage-linked CDOs, collected steady premiums, and held **almost no reserves** | Confidence was weak for years, so its CDS spread already sat above peers' |
| As house prices fell, the securities lost value, AIG was downgraded, and contracts required it to **post cash collateral** | US regional bank failures made investors nervous about weaker banks everywhere |
| It couldn't find the cash. On **16 Sep 2008** the Fed lent **\$85 billion** for ~80% stake; total support **> \$180 billion** | CDS spread jumped as shares fell and depositors withdrew; UBS takeover announced **19 March 2023** |
| **Lessons:** premiums looked like free money until the correlated event arrived; counterparty risk is **systemic**; **collateral calls turned a paper loss into a cash crisis**, so liquidity risk matters as much as credit risk | **Lessons:** the spread is a **live sensor** (ratings move far slower); fear is **self-reinforcing** (positive feedback loop); some bondholders were wiped out, so even "senior-sounding" debt is only as safe as its structure. Circuit breaker = central bank liquidity support |

---

## 5. CDOs and the 2008 Financial Crisis

### 5A. CDOs: securities made from securities

1. Thousands of mortgages are pooled into an MBS and cut into tranches.
2. Senior layers sell easily; the middle (**BBB**) layers are harder to sell.
3. Many BBB layers from different MBS are pooled into a **CDO (Collateralised Debt Obligation)** and tranched **again**.
4. The CDO's senior layer is protected by the layers below, so a large share was rated **AAA, even though every piece inside was BBB**.

**Why it was fragile:** the hidden assumption was that the BBB pieces would fail **independently** (each 5% chance → losing more than a fifth at once is very unlikely). In reality they all held loans exposed to the **same national housing market**, so they failed together. The "safe" senior CDO layers lost large amounts. **A second layer of tranching multiplied the model's error.**

> 💡 **In simple words:** BBB (kamzor) tukdon ko jod ke AAA bana diya, yeh maan ke ki sab ek saath fail nahi honge. Par sab ek hi housing market pe tike the, to sab ek saath gire.

### 5B. How the crisis chained together

1. **Cheap money:** low rates raise housing demand; prices climb and everyone expects more.
2. **Subprime loans:** weak borrowers get low "teaser" rates that reset higher after two years, assuming they can refinance.
3. **Sell it onward:** lenders are paid per loan sold, so quality costs them nothing (**broken feedback**, "originate-to-distribute").
4. **MBS and CDOs:** loans repackaged twice; ratings assume independent failures, so many tranches get AAA.
5. **Leverage and CDS:** banks borrow heavily to hold them; insurers sell protection with thin reserves.
6. **Prices fall:** resets hit, defaults rise together, losses cannot be traced. **Lehman fails on 15 September 2008.**

**Feedback loop:** falling prices → defaults → forced sales → prices fall further. Positive feedback drives the system away from stability.

### 5C. Timeline

| When | Event | Why it mattered |
|---|---|---|
| 2006 | US house prices peak and start to fall | The shared cause begins |
| Aug 2007 | BNP Paribas freezes three funds exposed to US mortgages | First sign nobody can value the securities |
| Sep 2007 | Run on Northern Rock (UK) | Funding fear reaches depositors |
| Mar 2008 | Bear Stearns rescued with official support | A major bank nearly fails |
| 7 Sep 2008 | Fannie Mae and Freddie Mac placed under US government control | The mortgage backbone is rescued |
| 15–16 Sep 2008 | Lehman files for bankruptcy; AIG rescued | Panic goes global |
| 3 Oct 2008 | US passes a \$700 billion bank rescue program | Governments take the losses |

### 5D. 2008 as a failure analysis

| Failure type | Engineering parallel | 2008 case |
|---|---|---|
| Model risk | Assuming independent failures | AAA ratings on tranches built on correlated loans |
| Hidden coupling | Unmapped dependencies | CDOs and CDS tied firms together invisibly |
| No fault isolation | A cascade with no circuit breaker | US subprime losses spread worldwide |
| Broken feedback | A supplier with no warranty cost | Originate-to-distribute lending |
| Thin safety factor | Designing with almost no margin | Banks with high leverage and little equity |

### 5E. Two real examples

**One subprime loan, traced:** a US family borrows \$300,000 on a **2/28** adjustable loan: 6% for two years (≈ **\$1,800/month**), then a reset. At an illustrative 9.5%, the payment rises to ≈ **\$2,490 (+38%)**. With prices falling they can't refinance, so they default. Their loan sat in an MBS pool, whose BBB layer went into a CDO, whose AAA layer AIG insured. One default is tiny; millions in the same pools broke the chain.

**Lehman, the 72-hour cascade:**
- *Before:* huge mortgage-related positions, financed mostly with **short-term borrowing rolled over every few days**, at very high leverage. As losses mounted, lenders stopped rolling it over; a rescue deal collapsed over 13–14 Sep 2008.
- **15 Sep:** bankruptcy, the largest in US history (> \$600 billion in assets).
- **16 Sep:** a money market fund holding Lehman debt **"breaks the buck"** (falls below \$1/unit); investors rush out of similar funds.
- **Next days:** short-term funding freezes worldwide; the shock reaches India as foreign investors sell. **Sensex fell from ~21,000 (Jan 2008) to ~8,000 (Oct 2008).**
- *Engineering parallel:* a structure that stays up only while a support is continuously renewed. A run is a structural failure mode.

**What changed after:** Basel III capital rules, Dodd-Frank, risk retention for securitisers, and bank stress tests.

---

## 6. Market-Linked Notes (MLNs)

A **debt instrument whose return depends on a market variable** such as the Nifty 50. In India: **market-linked debentures (MLDs)**. A principal-protected note = **zero-coupon bond + options**.

### 6A. How your ₹10 lakh for 5 years is split

| Part | Amount | Purpose |
|---|---|---|
| Zero-coupon bond at 7% | 10 ÷ 1.07⁵ ≈ **₹7.13 lakh** | Grows to ₹10 lakh at maturity → **principal protection** |
| Options on the Nifty | 10 − 7.13 ≈ **₹2.87 lakh** | Pay extra if the Nifty rises; expire worthless if it falls |

**Trade-off:** you cannot have full upside **and** full protection, because the protection uses up most of the money. Protection also depends on the **issuer paying**.

### 6B. Payoff formula and terms

$$\text{Maturity value} = \text{Principal} \times \big(1 + \text{Participation} \times \max(0,\ \text{Index return})\big)$$

(If there is a **cap**, the bracketed return can't exceed the cap.)

| Term | Meaning | Effect on you |
|---|---|---|
| Underlying | The index/basket tracked, e.g. Nifty 50 | What you're betting on |
| Tenor | Length of the note, e.g. 5 years | Money locked in |
| Participation rate | Share of the gain you get, e.g. 70% | Lower rate = less upside |
| Cap | Maximum return, if any | Limits your best case |
| Protection level | Share of principal returned if the market falls | Sets your worst case, **if the issuer pays** |

**Why participation is below 100%:** the options cost money. More protection → less money left for options → less upside.

### 6C. Solved example: ₹10 lakh, 5 years, 70% participation

| Nifty after 5 years | Note return | Value at maturity |
|---|---|---|
| Up 50% | 0.7 × 50% = 35% | ₹13.5 lakh |
| Flat | 0% | ₹10 lakh |
| Down 20% | 0%, principal protected | ₹10 lakh |
| Issuer defaults | Protection fails | Part or all of principal at risk |

**Compare:** a 7% fixed return for five years grows ₹10 lakh to ≈ **₹14.03 lakh** (10 × 1.07⁵), which **beats the note even when the Nifty rises 50%**.

> ⚠️ Slide says 35% over five years is "about 6% a year simple". Simple = 35 ÷ 5 = **7% a year**; about 6% (6.19%) is the **compound** annual rate.

**Real cases:** a Nifty-linked debenture (above); **Lehman minibonds**, sold to retail investors in Hong Kong and Singapore as "safe income", were worth a fraction of face value after Lehman failed.

### 6D. Risks and a buyer's checklist

| Risks | Five questions before buying |
|---|---|
| **Issuer credit risk:** protection fails if the issuer fails (Lehman) | 1. Who is the issuer, and how strong is its credit? |
| **Opportunity cost:** capped/partial upside can trail a plain FD | 2. Can I draw the payoff for a rise, a flat market and a fall? |
| **Illiquidity:** selling early may be hard or at a poor price | 3. What are the cap, participation rate and total fees? |
| **Hidden fees:** costs sit inside the structure, not on a bill | 4. Can I exit early, and at what price? |
| **Complexity:** payoff rules can hide how often you earn nothing | 5. Would a simple alternative give a similar result? |

---

## 7. Quick Revision 🧠

- **Securitisation:** loans → SPV → tranches → investors. Benefits: liquidity, capital recycling, diversification, choice. Risks: moral hazard, opacity, correlation, complexity.
- **Cash waterfall:** senior paid first, equity last. **Loss waterfall:** equity loses first, senior last (**subordination** = main credit enhancement).
- **Diversification works only for independent failures**, not common-mode failure.
- **Agency MBS:** credit risk removed by guarantee → **prepayment risk** appears. **India PTCs:** RBI requires risk retention.
- **Credit spread** = company yield − govt yield. **Implied PD ≈ spread ÷ loss-given-default.**
- **CDS premium** = notional × spread. **Payout** = notional × (1 − recovery). Reference entity is not a party; naked CDS exists; **counterparty risk** can make protection worthless.
- **CDO:** BBB tranches re-pooled into AAA, assuming independence → failed together.
- **2008 chain:** cheap money → subprime → originate-to-distribute → MBS/CDO → leverage + CDS → prices fall → Lehman (15 Sep 2008).
- **MLN** = zero-coupon bond + options. Value = P × (1 + participation × max(0, index return)). Protection is only as good as the issuer.

---

## 8. MCQs (Mid–Advanced Level) 📝

**Q1.** In the ₹500 crore pool (senior ₹400 cr at 7.5%, mezzanine ₹70 cr at 11%, equity ₹30 cr gets the residual), borrowers pay only ₹36 crore of interest this year. How is it distributed?
- A) Senior ₹30 cr, Mezzanine ₹6 cr, Equity ₹0
- B) Senior ₹28.8 cr, Mezzanine ₹5.04 cr, Equity ₹2.16 cr
- C) Senior ₹30 cr, Mezzanine ₹7.7 cr, Equity −₹1.7 cr
- D) Senior ₹30 cr, Mezzanine ₹3 cr, Equity ₹3 cr

> ✅ **Answer: A.** Cash flows down in strict order: senior takes its full ₹30 cr, mezzanine gets the remaining ₹6 cr (₹1.7 cr short of its ₹7.7 cr), and equity gets nothing. Option B shares cash pro-rata, which a waterfall does not do; a tranche can't receive a negative payment as in Option C. 📖 Section 2C

**Q2.** Using the same pool structure, the pool suffers a credit loss of ₹120 crore. How much does the senior tranche lose?
- A) ₹90 crore
- B) ₹20 crore (5% of the senior tranche)
- C) ₹0
- D) ₹96 crore (pro-rata share)

> ✅ **Answer: B.** Equity absorbs the first ₹30 cr and mezzanine the next ₹70 cr (₹100 cr in total). The remaining ₹20 cr hits senior: 20 ÷ 400 = 5%. Option D wrongly spreads losses pro-rata (400/500 × 120). 📖 Section 2D

**Q3.** When the pool earns its full ₹45 crore, what is the equity tranche's approximate annual return?
- A) 15%
- B) 9%
- C) 11%
- D) 24%

> ✅ **Answer: D.** Equity gets 45 − 30 − 7.7 = ₹7.3 cr on ₹30 cr = 24.3%. It earns the most because it takes losses first. Option B is just the pool's loan rate. 📖 Section 2C

**Q4.** "Subordination" in a securitisation refers to:
- A) The SPV being legally subordinate to the originating bank
- B) Senior tranches being paid after junior tranches
- C) Junior tranches absorbing losses before senior tranches, protecting the seniors
- D) Investors' claims ranking below the servicer's fees

> ✅ **Answer: C.** Losses run in reverse order (equity → mezzanine → senior). This loss-absorbing cushion is the main credit enhancement in a securitisation. Option B reverses the cash waterfall. 📖 Section 2D

**Q5.** A pool of 1,000 loans each has a 3% chance of default. Under which condition is the senior tranche most likely to suffer losses?
- A) The pool size is doubled to 2,000 independent loans
- B) A nationwide house-price fall raises every borrower's default chance at the same time
- C) Defaults are independent and spread across many cities
- D) The servicer charges a slightly higher fee

> ✅ **Answer: B.** This is common-mode failure: a shared cause makes defaults move together, so pool losses can far exceed the equity and mezzanine cushions. Independent defaults (Option C, Option A) cluster near 3% and are absorbed by the equity tranche. 📖 Section 2E

**Q6.** In a Ginnie Mae-guaranteed agency MBS, which risk replaces credit risk as the investor's main concern?
- A) Currency risk
- B) Equity market risk
- C) Counterparty risk of the servicer
- D) Prepayment risk

> ✅ **Answer: D.** The government guarantee removes credit risk. But when rates fall, homeowners refinance and investors get principal back early, which must be reinvested at lower rates. 📖 Section 2F

**Q7.** Why did post-2008 RBI rules require originators of Indian PTCs to retain part of the risk?
- A) To remove the need for credit ratings
- B) To reduce moral hazard, so the seller still cares about borrower quality
- C) To increase the originator's tax liability
- D) To make PTCs eligible for stock exchange listing

> ✅ **Answer: B.** If a lender sells every loan onward and keeps nothing, it has little reason to screen borrowers (moral hazard). Risk retention keeps "skin in the game". 📖 Sections 2B, 2F

**Q8.** A company's 5-year bond yields 10.25% and a comparable government bond yields 7%. What is the credit spread?
- A) 3.25 bp
- B) 32.5 bp
- C) 325 bp
- D) 46 bp

> ✅ **Answer: C.** 10.25% − 7% = 3.25% = 325 basis points (1 bp = 0.01%). Option B and Option A misplace the decimal. 📖 Sections 1, 3

**Q9.** A bond's credit spread is 3% and investors expect to recover 50% in a default. What yearly default probability does the market imply (rule of thumb)?
- A) 3%
- B) 1.5%
- C) 6%
- D) 5%

> ✅ **Answer: C.** Loss if default = 1 − 0.50 = 50%. Implied PD = 3% ÷ 0.5 = 6%. Option B multiplies instead of dividing; Option D divides by the recovery-adjusted 60% used in the slides, which doesn't apply here. 📖 Section 3

**Q10.** A fund buys CDS protection on ₹250 crore of a company's bonds at a spread of 150 bp. What is the annual premium?
- A) ₹0.375 crore
- B) ₹1.5 crore
- C) ₹37.5 crore
- D) ₹3.75 crore

> ✅ **Answer: D.** Premium = notional × spread = 250 × 1.5% = ₹3.75 cr a year. Option C uses 15%, and Option A uses 0.15%. 📖 Section 4

**Q11.** A bank holds CDS protection on ₹100 crore notional. The reference company defaults and bondholders recover 25%. Under cash settlement, what does the protection seller pay?
- A) ₹25 crore
- B) ₹60 crore
- C) ₹100 crore
- D) ₹75 crore

> ✅ **Answer: D.** Cash settlement pays face value − recovered amount = 100 × (1 − 0.25) = ₹75 cr. Option A is the recovery amount, and Option B uses the slides' 40% recovery instead. 📖 Section 4

**Q12.** Which statement about the reference entity in a CDS is correct?
- A) It is not a party to the contract and may not even know the CDS exists
- B) It receives the payout if it defaults
- C) It must approve the contract before trading
- D) It pays the premium to the protection seller

> ✅ **Answer: A.** The contract is between the protection buyer and seller. The reference entity is only the company, country or security being insured. 📖 Section 4

**Q13.** A hedge fund buys CDS protection on a company whose bonds it does not own. This is called a:
- A) Naked CDS
- B) Covered CDS
- C) Pass-through certificate
- D) Credit-linked note

> ✅ **Answer: A.** A "naked" CDS needs no actual loss to collect; it is effectively a bet on the borrower's decline. This surprised many people in 2008. 📖 Section 4

**Q14.** What turned AIG's CDS losses from a "paper loss" into a cash crisis in September 2008?
- A) After a downgrade, contracts required AIG to post cash collateral it did not have
- B) A sudden change in US tax law
- C) Its CDS buyers refused to pay premiums
- D) AIG was forced to buy back its own shares

> ✅ **Answer: A.** As the insured CDO layers lost value and AIG was downgraded, collateral calls demanded cash. With almost no reserves, AIG needed a \$85 billion Fed loan. Liquidity risk mattered as much as credit risk. 📖 Section 4

**Q15.** The Credit Suisse episode of March 2023 is best described as:
- A) A case where credit ratings warned investors well before the CDS market
- B) A default triggered by a CDS seller's failure
- C) A positive feedback loop: a wider spread raised funding costs and fear, widening the spread further
- D) A negative feedback loop that stabilised the bank's funding

> ✅ **Answer: C.** The CDS spread acted as a live sensor and fear fed on itself until UBS took over. Ratings moved far more slowly than the spread, which rules out Option A. 📖 Section 4

**Q16.** How did CDOs end up with large AAA-rated tranches even though every piece inside was rated BBB?
- A) Models assumed the BBB pieces would fail independently, so the senior layer looked very safe
- B) The CDO pools held only prime borrowers
- C) The underlying mortgages were government-guaranteed
- D) Rating agencies ignored subordination

> ✅ **Answer: A.** Under independence, losing more than a fifth of the pieces at once looked very unlikely. In reality all were exposed to the same housing market, so they failed together; a second round of tranching multiplied the error. 📖 Section 5A

**Q17.** In the 2008 failure analysis, "originate-to-distribute" lending is an example of:
- A) No fault isolation: losses spread worldwide
- B) Broken feedback: lenders were paid per loan sold, so poor quality cost them nothing
- C) Model risk: AAA ratings on correlated loans
- D) Thin safety factor: banks held too little equity

> ✅ **Answer: B.** Like a supplier with no warranty cost, the originator didn't bear the consequences of bad loans. Option D describes leverage, Option A contagion, and Option C the rating models. 📖 Section 5D

**Q18.** Why did Lehman's bankruptcy freeze short-term funding markets?
- A) Fannie Mae and Freddie Mac stopped buying mortgages
- B) The Fed raised interest rates the same day
- C) Lehman relied on short-term borrowing; its failure caused losses for money market funds (one "broke the buck"), so lenders stopped lending short-term
- D) Lehman's shares were removed from the stock exchange

> ✅ **Answer: C.** A money market fund holding Lehman debt fell below \$1 per unit, investors rushed out of similar funds, and short-term funding dried up worldwide. 📖 Section 5E

**Q19.** A 2/28 adjustable-rate loan of \$300,000 pays about \$1,800 a month at 6%. After the reset, the payment rises to about \$2,490. By roughly what percentage did the payment rise?
- A) 28%
- B) 38%
- C) 9.5%
- D) 50%

> ✅ **Answer: B.** (2,490 − 1,800) ÷ 1,800 ≈ 38%. With falling prices the family couldn't refinance and defaulted. Option C is the new interest rate, not the payment change. 📖 Section 5E

**Q20.** An issuer builds a 5-year principal-protected note of ₹10 lakh when the zero-coupon rate is 8%. Roughly how much is left to buy options?
- A) ₹2.87 lakh
- B) ₹0.8 lakh
- C) ₹6.81 lakh
- D) ₹3.19 lakh

> ✅ **Answer: D.** Bond portion = 10 ÷ 1.08⁵ = ₹6.81 lakh; options = 10 − 6.81 = ₹3.19 lakh. Option A is the slide's figure at 7%, and Option C is the bond portion. Higher rates make protection cheaper, leaving more for upside. 📖 Section 6A

**Q21.** A ₹20 lakh market-linked note has 60% participation and no cap. The Nifty rises 40% over the tenor, and the issuer pays in full. What is the maturity value?
- A) ₹28 lakh
- B) ₹24.8 lakh
- C) ₹20 lakh
- D) ₹22.4 lakh

> ✅ **Answer: B.** Return = 0.6 × 40% = 24%, so value = 20 × 1.24 = ₹24.8 lakh. Option A gives 100% participation. 📖 Section 6B

**Q22.** A ₹12 lakh note has 80% participation and a 15% cap on the note's return. The Nifty rises 25%. What is the maturity value?
- A) ₹13.8 lakh
- B) ₹13.92 lakh
- C) ₹15 lakh
- D) ₹14.4 lakh

> ✅ **Answer: A.** Uncapped return = 0.8 × 25% = 20%, but the cap limits it to 15%: 12 × 1.15 = ₹13.8 lakh. Option D ignores the cap, and Option C applies the full index return. 📖 Section 6B
