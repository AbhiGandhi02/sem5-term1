# Session 13 & 14: Financial Derivatives

> 🎓 **Recap:** Sessions 9–11 priced the underlying assets themselves (cost of capital, bonds, stocks, valuation). Now we study contracts whose value **comes from** those assets.

**After these two sessions you will be able to:**
- Explain what a derivative is and why hedgers, speculators and arbitrageurs use them
- Compare forwards, futures, options and swaps
- Price a forward with cost-of-carry and spot an arbitrage
- Read call/put payoffs, moneyness, break-evens and the Greeks
- Understand the Black-Scholes model at a working level (optional module)
- Build hedges (futures, protective put, currency forward/option) and income strategies (covered call/put)

---

## 1. What is a Derivative?

A **derivative** is a contract whose value **derives from an underlying**: a stock, an index (Nifty 50), a commodity, a currency (USD/INR) or an interest rate.

> 🛠️ **Engineering analogy:** a derivative is a function `f(S)`. Input = underlying price, output = payout. You hold the *rule*, not the asset, like a strain gauge reporting on a bridge.

| Purpose | Meaning | Typical user |
|---|---|---|
| **Hedging** | Transfer an existing risk to someone else | Exporters/importers (currency), airlines (fuel), farmers (crops) |
| **Speculation** | Take a view with leverage | Traders |
| **Arbitrage** | Exploit price gaps for riskless profit | Funds, banks |

### 1A. Key vocabulary

| Term | Meaning |
|---|---|
| Underlying | Asset the contract is based on |
| Strike (K) | Agreed price at which the deal happens |
| Premium | Price paid by the option buyer for the right |
| Expiry | Date the contract ends |
| Lot size | Fixed quantity per contract |
| Long / Short | Buyer / seller of the contract |
| Margin | Deposit that covers potential losses |
| Exchange vs OTC | Standardised and cleared vs privately negotiated |

### 1B. What can a derivative track?

| Class | Example | Typical use | Settlement |
|---|---|---|---|
| Equity | Reliance, Infosys | Hedge or income on holdings | Cash (shares in some cases) |
| Index | Nifty 50, Bank Nifty | Portfolio hedge, market view | Cash |
| Currency | USD/INR | Exporter/importer hedge | Cash in INR |
| Commodity | Gold, crude oil | Producers/buyers lock prices | Cash or physical |
| Interest rate | Bond yields | Banks manage borrowing cost | Cash |

**Cash settlement:** index options pay only the rupee difference; nobody delivers 50 stocks.

### 1C. Exchange trading: clearing house and margin
- **Novation:** once a trade is matched, the **clearing corporation** becomes the buyer to every seller and the seller to every buyer. You never face the other trader, so default risk is tiny.
- **Margin:** a good-faith deposit. **Initial margin** at entry + **MTM margin** for daily losses. If your balance falls below the **maintenance level**, you get a **margin call**.
- **Leverage (illustrative):** 1 Nifty lot = 24,000 × 75 = **₹18 lakh** exposure. At ~10% margin you post ₹1.8 lakh. A 1% index move = ₹18,000 = **10% of your margin**.

> 💡 **In simple words:** Margin ek chhota deposit hai jo bade exposure ko control karta hai. Isliye 1% market move tumhare paise ka 10% hila deta hai: yahi leverage hai, faayda bhi bada, nuksaan bhi bada.

---

## 2. Forwards, Futures and Options Compared

| Feature | Forward | Future | Option |
|---|---|---|---|
| Venue | OTC | Exchange | Exchange or OTC |
| Terms | Customised | Standardised | Standardised |
| Obligation | Both sides | Both sides | Buyer: right; seller: obligation |
| Upfront | None | Margin | Premium (buyer) |
| Settlement | At maturity | Daily mark-to-market | At expiry or exercise |
| Counterparty risk | High | Very low (clearing house) | Very low on exchange |
| Buyer max loss | Unlimited | Unlimited | Premium only |

Forwards/futures have **linear** payoffs. Options have **non-linear** (bent) payoffs.

### 2A. Forward example: farmer and flour mill
Farmer expects **100 tonnes** of wheat; the mill fears price rises. Forward at **₹25,000/tonne**.

| Spot at harvest | Farmer (vs market) | Mill (vs market) |
|---|---|---|
| ₹22,000 | +₹3 lakh | −₹3 lakh |
| ₹25,000 | ₹0 | ₹0 |
| ₹28,000 | −₹3 lakh (forgone) | +₹3 lakh |

- **Zero-sum:** one side gains exactly what the other loses relative to spot.
- The real purpose is **certainty** for both.
- **Default risk:** at ₹28,000 the farmer is tempted to sell elsewhere. Exchanges fix this with margin and daily settlement.

### 2B. Currency forward for an importer
Machine costs **USD 500,000**, payable in 3 months. Spot ₹88.00; 3-month forward ₹89.00.

| Spot in 3 months | Cost, no hedge | Cost with forward | Forward vs spot |
|---|---|---|---|
| ₹91.00 | ₹4.55 cr | ₹4.45 cr | Saves ₹10 lakh |
| ₹89.00 | ₹4.45 cr | ₹4.45 cr | ₹0 |
| ₹87.00 | ₹4.35 cr | ₹4.45 cr | Costs ₹10 lakh extra |

- **Why:** the firm quoted its customer in rupees, so fixing the import cost protects its margin.
- **Forward points** = 89.00 − 88.00 = ₹1.00. This reflects the **interest-rate gap** between India and the US, **not a forecast**.
- Customised amount and date, no upfront premium; early exit needs the bank's agreement.

### 2C. Forward pricing: cost-of-carry
A forward/futures price is set by **no-arbitrage**, not opinion:

$$F = S \times (1 + r \times T) - \text{dividends}$$

| Input | Value |
|---|---|
| Spot S | ₹1,400 |
| r | 6.5% a year |
| T | 1 month = 1/12 |
| Fair futures price | 1,400 × (1 + 0.065/12) = **₹1,407.58** |
| With ₹10 dividend this month | 1,407.58 − 10 = ₹1,397.58 |

| Future trades at… | Arbitrage | Profit |
|---|---|---|
| ₹1,415 (too high) | **Cash-and-carry:** borrow, buy stock at 1,400, sell future at 1,415; deliver and repay 1,407.58 | ≈ ₹7.42/share |
| ₹1,400 (too low) | **Reverse:** short stock, invest proceeds, buy future | ≈ ₹7.58/share |

Arbitrageurs push the price back to fair value. Real costs keep small gaps alive.

> 💡 **In simple words:** Future ka price "andaaza" nahi hai. Aaj stock khareedo aur T time tak hold karo, uska kharcha (interest) jodo, dividend ghatao: wahi fair future price hai. Isse upar/neeche gaya to free profit milta hai, jo jaldi khatam ho jaata hai.

### 2D. Futures: linear payoff and mark-to-market
Long 1 Nifty future at 24,000, lot 75: **Profit = (Close − 24,000) × 75**. At 24,100: +₹7,500; at 23,900: −₹7,500. Loss is unlimited on either side.

**Daily MTM:**

| Day | Settlement | Change | Day P/L | Cumulative |
|---|---|---|---|---|
| 1 | 24,100 | +100 | +₹7,500 | +₹7,500 |
| 2 | 24,040 | −60 | −₹4,500 | +₹3,000 |
| 3 | 23,900 | −140 | −₹10,500 | −₹7,500 |
| 4 | 24,000 | +100 | +₹7,500 | ₹0 |

On Day 3, ₹10,500 leaves your account. Below maintenance margin you must top up or the position is closed. Final P/L = what a single settlement at 24,000 gives: **zero**.

### 2E. Index vs stock futures, basis and rollover

| | Index futures | Stock futures |
|---|---|---|
| Underlying | Nifty 50, Bank Nifty | Single shares (Reliance) |
| Settlement | Cash | **Physical delivery** in India |
| Used for | Portfolio hedge, market view | Stock-specific view/hedge |
| Lot size | Set by exchange (Nifty: 75) | Set per stock |

- **Basis = Futures − Spot.** Spot 24,000, near-month 24,080 → basis 80. It **shrinks to zero at expiry**.
- **Rollover:** close near month, open next. Sell near at 24,060, buy next at 24,130 → roll cost 70 × 75 = **₹5,250**.
- Expiry dates are set by the exchange and change over time; always check current rules.

### 2F. Hedging a portfolio with index futures
₹10 crore portfolio tracking Nifty. Contract value = 24,000 × 75 = ₹18 lakh → lots = 10 cr ÷ 18 lakh ≈ **56 lots** sold.

| Nifty | Portfolio | 56 short futures | Net |
|---|---|---|---|
| −5% (22,800) | −₹50 lakh | +₹50.4 lakh | ≈ ₹0 |
| +5% (25,200) | +₹50 lakh | −₹50.4 lakh | ≈ ₹0 |

A futures hedge removes risk in **both** directions. Use **put options** to keep the upside.

### 2G. Gold forward vs gold future (same trade)
Jeweller needs **1 kg gold** in 3 months; locks ₹1,01,000 per 10 g (₹1.01 crore).

| Feature | Forward with a bank | Gold future |
|---|---|---|
| Contract | Custom weight/date | Standard lot/expiry |
| Counterparty | Bank (credit risk) | Clearing house |
| Cash flows | At maturity | Daily MTM; margin ≈ ₹5 lakh |
| Exit early | Negotiate with bank | Reverse any time |
| Price → ₹1,10,000 | Gain ₹9 lakh at maturity | Same ₹9 lakh, collected daily |
| Price → ₹95,000 | Loss ₹6 lakh at maturity | Same ₹6 lakh, paid daily via margin calls |

---

## 3. Swaps: Floating to Fixed

A **swap** exchanges cash flows on a **notional** amount. A company has a ₹100 crore loan at **MIBOR + 1%**. It enters a swap: **pays 7% fixed, receives MIBOR**.

| MIBOR | Loan interest | Swap net | Total cost |
|---|---|---|---|
| 6% | ₹7 cr | pay 7, get 6 = −₹1 cr | ₹8 cr = **8%** |
| 8% | ₹9 cr | pay 7, get 8 = +₹1 cr | ₹8 cr = **8%** |
| 5% | ₹6 cr | pay 7, get 5 = −₹2 cr | ₹8 cr = **8%** |

- **Result:** floating cost becomes a fixed **8%** (= 7% fixed + 1% spread). Budget certainty, but no gain if rates fall.
- **Other swaps:** currency swaps (foreign loans), commodity swaps (airlines locking fuel).
- Swaps are OTC, so counterparty risk matters. Only **net interest** is paid; the notional is not exchanged.

> 💡 **In simple words:** MIBOR jitna bhi ho, loan pe jo MIBOR dena hai woh swap se wapas mil jaata hai. Bacha sirf fixed 7% + loan ka 1% spread = 8%, har haal mein.

---

## 4. Options

An option gives its buyer the **right but not the obligation** to buy or sell the underlying at strike **K** on or before expiry, in return for a **premium**.

| Buyer (long) | Seller / writer (short) |
|---|---|
| Pays premium | Receives premium |
| Chooses whether to exercise | Must honour if exercised |
| Loss limited to premium | Margin required; risk can be large |

### 4A. Everyday options
- **ESOP = call option:** 1,000 options, exercise price ₹500. Payoff = max(S − 500, 0) × 1,000. At ₹900 → ₹4,00,000; at ₹500 or ₹450 → ₹0 (not exercised). Downside is zero.
- **Car insurance = put option:** car insured for ₹8 lakh, premium ₹20,000. Total loss → claim ₹8,00,000; car worth ₹5 lakh → claim ≈ ₹3,00,000; no accident → premium lost.

### 4B. Calls and puts

| | Call | Put |
|---|---|---|
| Right to | Buy at K | Sell at K |
| Buyer expects | Price to rise | Price to fall |
| Value at expiry | max(S − K, 0) | max(K − S, 0) |
| In-the-money (ITM) | S > K | S < K |
| At-the-money (ATM) | S ≈ K | S ≈ K |
| Out-of-the-money (OTM) | S < K | S > K |
| Analogy | Token advance on a flat | Car insurance |

**Activity:** K = 100, premium 5, spot at expiry 112. Call buyer profit = 12 − 5 = **₹7**. Put buyer profit = 0 − 5 = **−₹5**.

### 4C. The seller side
Sell 1 lot Nifty 24,500 call at premium 100 (lot 75) → receive **₹7,500**.

| Nifty at expiry | Call value | Writer P/L per unit | On 75 |
|---|---|---|---|
| 24,300 | 0 | +100 | +₹7,500 |
| 24,600 | 100 | 0 | ₹0 (break-even) |
| 24,800 | 300 | −200 | −₹15,000 |
| 25,500 | 1,000 | −900 | −₹67,500 |

**Asymmetry:** gain capped at the premium; loss grows without limit. That is why writers post margin and why a **naked short call** is dangerous.

### 4D. Moneyness on an option chain (Nifty spot 24,000)

| Strike | Call premium | Call status | Intrinsic + time | Put premium | Put status |
|---|---|---|---|---|---|
| 23,800 | 290 | ITM | 200 + 90 | 60 | OTM |
| 24,000 | 150 | ATM | 0 + 150 | 140 | ATM |
| 24,200 | 70 | OTM | 0 + 70 | 280 | ITM (200 + 80) |

- **Premium = Intrinsic value + Time value.**
- ATM options carry the most time value; deep ITM options are mostly intrinsic.
- OTM options are cheap but need a large move, so they usually expire worthless.

### 4E. American vs European
- **European:** exercise **only at expiry**. NSE index options are European.
- **American:** exercise **any time** up to expiry (common for US stock options). Not about geography.
- **Premium drivers:** spot vs strike, time to expiry, volatility, interest rates. Time value decays to zero at expiry (**theta decay**), which favours sellers.

### 4F. The Greeks in plain words

| Greek | Measures | Example |
|---|---|---|
| **Delta** | Premium change per ₹1 move in underlying | ATM call ≈ 0.5 → Nifty +100 lifts premium ≈ 50 |
| **Theta** | Daily time decay | Call losing ₹4/day hurts buyers, helps sellers; speeds up near expiry |
| **Vega** | Sensitivity to volatility | Elections/results raise all premiums |
| **Rho** | Sensitivity to interest rates | Minor for short-dated options |

**Worked example:** ATM call 150, delta 0.5. Nifty +100 → premium ≈ **200**. Nothing moves for 5 days at theta −4 → premium ≈ **130**.

---

## 5. Optional Module: Black-Scholes and the Greeks in Depth

### 5A. Core idea
Black and Scholes (1973), with Merton, showed a European option can be priced by **copying its payoff** with the underlying plus borrowing. By no-arbitrage, the option must cost what the copy costs (Scholes and Merton: 1997 Nobel Prize).
- **Replication:** hold Δ units of the index, financed by borrowing; rebalance so the random part cancels.
- **Risk-neutral pricing:** value = discounted expected payoff, with the index growing at r − q.
- **The surprise:** the **expected market return does not appear** in the formula. Only volatility matters.
- **SPX 7,700 call as a copy:** hold 0.5375 index units (4,120.7) financed by borrowing 3,846.4 → 4,120.7 − 3,846.4 = **274.4**, the Black-Scholes price.

**Price process:** dS/S = μ dt + σ dW. The price at expiry is **lognormal** (always positive; returns normal). Variance grows with T, so spread grows with **√T**. Example: σ = 16%, T = 101/365 → σ√T = 0.0842; one-SD range ≈ **7,048 to 8,340** from 7,666.45 (full year ≈ 6,533 to 8,997).

### 5B. Seven assumptions

| # | Assumption | Reality and consequence |
|---|---|---|
| 1 | Lognormal prices (GBM) | Fat tails and skew; far OTM options mispriced |
| 2 | Constant volatility | Volatility moves (VIX) and differs by strike: the **smile**. Fix: Heston |
| 3 | Continuous price paths | Gaps/crashes break the hedge. Fix: jump-diffusion (Merton 1976) |
| 4 | Constant r and known dividend yield q | Minor for short options, larger for LEAPS |
| 5 | Frictionless markets | Spreads, margin, discrete rebalancing create hedging error |
| 6 | No arbitrage | Broadly true; model price is a reference point |
| 7 | European exercise | American options need binomial trees. **SPX options are European** |

**Real-world failures:** (1) Oct 19, 1987: S&P 500 fell ~20% in a day, >20 standard deviations at 16% vol (daily σ ≈ 1.0%), impossible under lognormal prices; markets now charge extra for OTM puts. (2) The **smile**: 16% used for the 7,700 call, 20% for the 7,200 put. (3) **Discrete hedging** leaves P&L noise (gamma risk).

### 5C. The formula

$$C = S e^{-qT} N(d_1) - K e^{-rT} N(d_2) \qquad P = K e^{-rT} N(-d_2) - S e^{-qT} N(-d_1)$$

$$d_1 = \frac{\ln(S/K) + (r - q + \sigma^2/2)T}{\sigma\sqrt{T}} \qquad d_2 = d_1 - \sigma\sqrt{T}$$

| Symbol | Meaning | Call example |
|---|---|---|
| S | Current index | 7,666.45 |
| K | Strike | 7,700 |
| T | Years to expiry | 101/365 = 0.2767 |
| r | Risk-free rate (continuous) | 4.5% |
| q | Dividend yield | 1.2% |
| σ | Volatility of log returns | 16% |
| N(x) | Standard normal cumulative probability | N(0.0987) = 0.5393 |

**Reading d1 and d2:** d1 = distance from index to strike in standard deviations, plus a growth adjustment. **N(d2) = risk-neutral chance of finishing above K.**

| Piece | Call 7,700 | Put 7,200 |
|---|---|---|
| ln(S/K) | −0.0044 | +0.0628 |
| (r − q + σ²/2)T | 0.0127 | 0.0106 |
| σ√T | 0.0842 | 0.0894 |
| d1 | 0.0987 | 0.8203 |
| d2 | 0.0145 | 0.7309 |

> 💡 **In simple words:** d1/d2 batate hain ki spot strike se kitne "standard deviation" door hai. Call almost ATM hai (d ≈ 0), isliye ~50% chance. Put ~0.8 SD OTM hai, isliye sirf ~23% chance.

**Sanity checks:** T → 0 gives intrinsic value; σ → 0 makes the option a forward; very large S → call ≈ S e^(−qT) − K e^(−rT); very small S → call → 0.
**Put-call parity:** C − P = S e^(−qT) − K e^(−rT). Check: 274.45 − 238.13 = 36.32 = 7,641.0 − 7,604.7 ✓.
**Rule of thumb:** ATM call ≈ 0.4 × S × σ√T = 0.4 × 7,666 × 0.0842 ≈ 258 (exact 274; gap = drift).

### 5D. Two CBOE SPX examples (European, cash-settled, \$100 multiplier)

| | Example 1: 7,700 call, Jan 15, 2027 | Example 2: 7,200 put, Dec 18, 2026 |
|---|---|---|
| S, T, σ | 7,666.45; 101/365; 16% | 7,666.45; 73/365 = 0.20; 20% (skew) |
| d1, d2 | 0.0987, 0.0145 | 0.8203, 0.7309 |
| Probabilities | N(d1) 0.5393, N(d2) 0.5058 | N(−d1) 0.2060, N(−d2) 0.2324 |
| Two pieces | 4,120.7 − 3,846.4 | 1,658.3 − 1,575.5 |
| **Price** | **≈ 274.4 pts ≈ \$27,445/contract** | **≈ 82.8 pts ≈ \$8,285/contract** |
| Chance ITM | 50.6% | ≈ 23% |
| Break-even | 7,700 + 274.4 = **7,974** (+4.0%) | 7,200 − 82.8 = **7,117** (−7.2%) |

Put insight: insurance costs ≈ 1.1% of spot. If the index fell 8% at once, the put would be worth ≈ 305, about **3.7×** its cost.

### 5E. Greeks in depth

| Greek | Definition | Call | Put |
|---|---|---|---|
| Delta Δ | ∂V/∂S | e^(−qT)N(d1) | −e^(−qT)N(−d1) |
| Gamma Γ | ∂²V/∂S² | e^(−qT)n(d1)/(Sσ√T) | same |
| Vega | ∂V/∂σ | S e^(−qT)n(d1)√T | same |
| Theta Θ | −∂V/∂T | (time decay; negative for long options) | |
| Rho ρ | ∂V/∂r | K T e^(−rT)N(d2) | −K T e^(−rT)N(−d2) |

**Delta (hedge ratio):** calls 0 to +1, puts −1 to 0; put delta = call delta − e^(−qT).

| Call (101 days, 16%) | 7,200 (ITM) | 7,700 (ATM) | 8,200 (OTM) |
|---|---|---|---|
| Price | 597.8 | 274.4 | 96.0 |
| Delta | 0.812 | 0.538 | 0.257 |

1 SPX call = 0.5375 × 100 = **53.75 index units** of exposure. Delta ≈ chance of finishing ITM (N(d1) 54% vs N(d2) 51%). Delta rises as the index rises (7,300 → 0.313; 8,000 → 0.725).

**Gamma (curvature):** rate of change of delta; highest for **ATM options near expiry**.

| Days to expiry | 101 | 60 | 30 | 7 | 1 |
|---|---|---|---|---|---|
| Price | 274.4 | 201.9 | 134.0 | 54.5 | 12.5 |
| Gamma | 0.0006 | 0.0008 | 0.0011 | 0.0023 | 0.0055 |
| Theta/day | −1.60 | −1.98 | −2.66 | −5.08 | −11.48 |
| Vega | 15.96 | 12.36 | 8.76 | 4.18 | 1.41 |

- **Long options = long gamma** (profit from big moves either way). **Short options = short gamma** (hurt by big moves, paid by time decay).
- **Theta** accelerates near expiry because time value scales with √T. "Theta pays for gamma": ½σ²S²Γ/365 ≈ **1.26** of the 1.60 daily decay.
- **Vega** peaks ATM and grows with √T. IV 16% → 19.5% lifts the call ≈ 3.5 × 15.96 ≈ **55.9** points. The 7,200 put has lower vega (9.75) because it is further OTM.
- **Rho:** higher rates help calls, hurt puts. Call 7,700: +10.64; Put 7,700: −10.40; Put 7,200 (73 days): −3.32.

**Cheat sheet (long positions; short positions flip every sign):**

| Greek | Long call | Long put | Largest when |
|---|---|---|---|
| Delta | + | − | Deep ITM |
| Gamma | + | + | ATM, near expiry |
| Theta | − | − | ATM, near expiry |
| Vega | + | + | ATM, long maturity |
| Rho | + | − | Long maturity |

> ⚠️ Slides 36, 40 and 41 write gamma as **0.00057** in their calculations, but the figures they show (e.g. gamma P&L +1.80, theta-pays-gamma 1.26) match gamma ≈ **0.00061**, which is what the model actually gives. The table's rounded "0.0006" is consistent with both.

### 5F. Greeks explain one day of P&L
Long SPX 7,700 call (274.45). Next day: index +1% (+76.66), implied vol −1 point, 1 day passes.

| Source | Formula | Points |
|---|---|---|
| Delta | 0.5375 × 76.66 | +41.21 |
| Gamma | ½ × Γ × 76.66² | +1.80 |
| Vega | 15.96 × (−1) | −15.96 |
| Theta | 1 day | −1.60 |
| **Sum of Greeks** | | **+25.45** |
| Full repricing | 300.09 − 274.45 | +25.64 |

A good index rise was largely offset by a fall in volatility: the classic answer to "why did my option not gain more?"

### 5G. Delta hedging a short position
Dealer **sells 10 SPX 7,700 calls** at 274.45 → collects 10 × 100 × 274.45 = **\$274,450**; short ≈ 537.5 index units of delta.

| Step | Action | Numbers |
|---|---|---|
| 1. Hedge | Buy index exposure = delta | ≈ 537.5 units (≈ \$4.12 mn) |
| 2. Index +100 | Calls lose ≈ \$56,600; hedge gains \$53,750 | Net ≈ −\$2,850 (**gamma loss**) |
| 3. Rebalance | Delta now 0.598 | Buy ≈ 60 more units |
| 4. Time passes | Theta earned 10 × 100 × 1.60 | ≈ +\$1,600/day |

**Break-even move:** ½Γ·ΔS² = |Θ| → ΔS ≈ √(2 × 1.60 / Γ) ≈ **75 points ≈ 1%**, about one daily SD at 16% vol. If the index moves more than implied volatility says, the short-option dealer loses.

### 5H. Implied volatility and skew
- **Implied volatility:** solve for the σ that matches the market price. If the 7,700 call traded at 330, IV would be **19.5%** (vs 16% assumed).
- **Skew:** OTM puts trade at higher IV than calls. One σ for all strikes is the model's weakness.
- **Other limits:** American options need trees; jumps and fat tails; VIX options use Black-76; rates/dividends are not constant. The **VIX** summarises 30-day implied volatility from SPX options.

---

## 6. Choosing the Right Tool

| Situation | Tool | Why |
|---|---|---|
| Farmer wants a fixed harvest price | Forward or commodity future | Price certainty |
| Importer pays USD in 3 months | Currency forward, or USD call | Forward fixes cost; call keeps benefit if rupee strengthens |
| Fund manager fears a dip | Short index futures or put | Futures remove both ways; put keeps upside |
| Airline worried about fuel | Futures or commodity swap | Locks input cost |
| Floating-rate loan | Interest rate swap | Fixed payments |
| Trader wants limited-loss bet | Buy a call or put | Loss capped at premium |
| Employee wants a floor on share value | Protective put | Downside protection |

**Practice cases (suggested answers):**
1. Steel maker buying coking coal in 2 months → risk: price rise → commodity future/forward (or call).
2. Software firm receiving EUR 2 mn in 90 days → risk: euro falls → sell EUR forward or buy EUR put.
3. ₹5 lakh of shares, fear a fall but want gains → **protective put**.
4. Startup with a floating-rate loan, rates rising → **interest rate swap** (pay fixed).
5. Bullish on Nifty, max loss ₹10,000 → **buy a call** with premium ≤ ₹10,000.
6. Farmer wants a fixed price from a trusted local trader → **forward**.

---

## 7. Payoffs and Motives (K = 100, premium 5)

| Spot | 80 | 90 | 95 | 100 | 105 | 110 | 120 |
|---|---|---|---|---|---|---|---|
| Long call | −5 | −5 | −5 | −5 | 0 | +5 | +15 |
| Long put | +15 | +5 | 0 | −5 | −5 | −5 | −5 |

(Short call and short put are the mirror images.)

| Position | Max profit | Max loss | Break-even | Motive |
|---|---|---|---|---|
| Long call | Unlimited | ₹5 | 105 | Bullish, limited risk |
| Short call | ₹5 | Unlimited | 105 | Mildly bearish, income |
| Long put | ₹95 | ₹5 | 95 | Bearish or insurance |
| Short put | ₹5 | ₹95 | 95 | Mildly bullish, buy cheaper |

**Leverage:** stock ₹100, call ₹5. At ₹120 the share gains 20%, the call gains 15/5 = **+300%**. At ₹100 the share is flat, the call loses **−100%**.

---

## 8. Hedging

**Hedging** = taking a derivative position that offsets an existing risk. It **reduces uncertainty; it does not maximise profit** (think factor of safety).
- **Forward hedge:** locks a price; upside surrendered.
- **Option hedge:** locks a worst case, keeps upside; costs a premium.
- **Portfolio hedge:** buy an OTM Nifty put.

### 8A. Protective put
Portfolio ₹18 lakh (24,000 × 75). Buy **23,500 put at 150** (cost ₹11,250).

| Nifty at expiry | Unhedged | Put payoff | Hedged net |
|---|---|---|---|
| 26,000 | +₹1,50,000 | 0 | +₹1,38,750 |
| 24,000 | 0 | 0 | −₹11,250 |
| 23,500 | −₹37,500 | 0 | −₹48,750 |
| 22,000 | −₹1,50,000 | +₹1,12,500 | −₹48,750 |

**Max loss = (24,000 − 23,500 + 150) × 75 = ₹48,750**, however far the market falls. The premium works like an insurance deductible.

### 8B. Limits of hedging
- **Cost drag:** rolling puts monthly erodes returns if markets don't fall.
- **Basis risk:** hedging a single-stock portfolio with Nifty puts leaves a gap.
- **Hedge ratio:** hedge only the exposure you are sure about.
- **Cautionary case:** exporter sells USD 1M forward at 88.90, order is cancelled, rupee weakens to 92 → forward loses (92 − 88.90) × 1M = **₹31 lakh** with nothing to offset. **The hedge became a speculation.**

### 8C. Real example 1: IT exporter
Bengaluru IT firm receives **USD 1,000,000** in 3 months; costs are in rupees. Risk: rupee **strengthens**. Spot ₹88.00, 3-month forward ₹88.90, USD put strike ₹88.50 premium ₹0.60 (₹6 lakh).

| Spot in 3 months | No hedge | Forward 88.90 | Put (net of premium) |
|---|---|---|---|
| 85.00 | ₹8.50 cr | ₹8.89 cr | ₹8.79 cr (floor) |
| 88.90 | ₹8.89 cr | ₹8.89 cr | ₹8.79 cr |
| 92.00 | ₹9.20 cr | ₹8.89 cr | ₹9.14 cr |

- **No hedge:** revenue swings ₹8.5–9.2 cr.
- **Forward:** ₹8.89 cr certain; gives up gains if the rupee weakens.
- **Option:** floor = (88.50 − 0.60) × 1M = ₹8.79 cr, upside retained; premium = insurance cost.
- **Importer (mirror case):** risk is the rupee **weakening** → buy forward or a USD call.

---

## 9. Covered Calls and Covered Puts

### 9A. Covered call
**Own the stock + sell a call.** Shares "cover" the delivery obligation. View: neutral to mildly bullish. Trade-off: premium income and a small cushion for **capped upside** (like a clipper circuit cutting off the top of a waveform).

**Lifecycle:** hold 500 shares at ₹1,400 → sell one-month ₹1,460 call (~4% OTM) for ₹25 → collect ₹12,500. Below 1,460 at expiry: keep shares + premium (yield 25/1,400 ≈ **1.8% for the month**, not guaranteed). Above 1,460: shares sold at 1,460 (**assignment**). **Rolling:** buy back and sell a later/higher strike.

| Price at expiry | Net/share | On 500 |
|---|---|---|
| 1,350 | −25 | −₹12,500 |
| 1,375 | 0 | break-even |
| 1,400 | +25 | +₹12,500 |
| 1,460 | +85 | +₹42,500 (max) |
| 1,600 | +85 | +₹42,500 |

- **Max profit** = (1,460 − 1,400) + 25 = **₹85/share**
- **Break-even** = 1,400 − 25 = **₹1,375**
- **Opportunity cost:** a rally to ₹1,600 still earns only ₹85.

### 9B. Covered put
**Short stock + sell a put.** Example: short at 1,400, sell 1,340 put for 20 → max profit = 60 + 20 = **₹80**; break-even = **1,420**; max loss **unlimited** on a rally.

| | Covered call | Covered put |
|---|---|---|
| View | Neutral to mildly bullish | Neutral to mildly bearish |
| Positions | Long stock + short call | Short stock + short put |
| Max loss | Large (stock to zero) | Unlimited |

### 9C. Strategy by market view

| View | Strategy | Risk | Reward |
|---|---|---|---|
| Strongly bullish | Long call | Premium | Unlimited |
| Mildly bullish | Short put / covered call | Large downside | Premium (+ capped gain) |
| Neutral | Covered call | Stock falls | Premium |
| Mildly bearish | Covered put | Unlimited rally loss | Premium + capped gain |
| Strongly bearish | Long put | Premium | Large |
| Protect holdings | Protective put / forward | Premium / lost upside | Loss floor |

### 9D. Risk management
**Common mistakes:** selling naked options for small premiums; ignoring lot size; forgetting time decay as a buyer; over-leveraging with margin; no exit plan on expiry day.
**Checklist:** define max loss first; size so one loss is survivable; know break-even and payoff shape; check margin; write down the exit rule.
SEBI studies report that a large majority of individual F&O traders lose money. Treat derivatives as **risk-management tools first**.

### 9E. Class quiz answers (slide 58)
Q1 Future · Q2 ₹7 (15 − 8) · Q3 ₹480 · Q4 European · Q5 Keeps upside if rupee weakens · Q6 ₹785 · Q7 Capped upside · Q8 K − premium.

---

## 10. Quick Revision 🧠

- Derivative = value from an underlying; uses: **hedging, speculation, arbitrage**
- Forwards/futures = **obligations, linear**; options = **rights, non-linear**
- Futures: exchange, standardised, **daily MTM**, clearing house removes default risk
- **F = S(1 + rT) − dividends**; any gap → cash-and-carry or reverse arbitrage
- **Basis = F − S → 0 at expiry**; rollover cost = (next − near) × lot
- Hedge lots = portfolio value ÷ (index × lot size)
- Swap: floating + (pay fixed, receive floating) = **fixed + spread**
- Call = max(S − K, 0); put = max(K − S, 0); **premium = intrinsic + time value**
- Long call BE = K + premium; long put BE = K − premium
- Greeks: Δ (price), Γ (curvature), Θ (time, negative for buyers), Vega (vol), ρ (rates)
- Black-Scholes: expected return doesn't matter; **N(d2) = P(ITM)**; parity **C − P = S e^(−qT) − K e^(−rT)**
- Protective put max loss = (S − K + premium) × lot
- Covered call: max profit = (K − cost) + premium; BE = cost − premium

---

## 11. MCQs (Mid–Advanced Level) 📝

**Q1.** Which feature best distinguishes a futures contract from a forward contract?
- A) Futures are settled daily through mark-to-market with a clearing house as counterparty
- B) Futures can be customised for any amount and date
- C) Futures give the buyer a right but not an obligation
- D) Futures require the buyer to pay a premium upfront

> ✅ **Answer: A.** Futures are standardised, exchange-traded and marked to market daily, with novation to the clearing house. Option C describes an option, Option D describes an option premium, and Option B describes a forward. 📖 Section 2

**Q2.** A stock trades at ₹2,000. The risk-free rate is 6% a year, and a ₹15 dividend is due within 3 months. Using simple cost-of-carry, what is the fair 3-month futures price?
- A) ₹2,045
- B) ₹2,030
- C) ₹1,985
- D) ₹2,015

> ✅ **Answer: D.** F = 2,000 × (1 + 0.06 × 0.25) − 15 = 2,030 − 15 = **₹2,015**. Option B ignores the dividend, Option A adds it instead of subtracting, and Option C subtracts the carry cost. 📖 Section 2C

**Q3.** In the same case (fair price ₹2,015), the 3-month future trades at ₹2,040. What is the arbitrage?
- A) Borrow, buy the stock and sell the future; riskless profit ≈ ₹25 per share
- B) Short the stock and buy the future; riskless profit ≈ ₹25 per share
- C) Buy the future only; profit ≈ ₹40 per share
- D) No arbitrage exists because futures prices reflect market opinion

> ✅ **Answer: A.** The future is overpriced, so do cash-and-carry: buy spot with borrowed money and sell the expensive future. Profit = 2,040 − 2,015 = **₹25**. Option B is the trade for an underpriced future, and Option D contradicts the no-arbitrage principle. 📖 Section 2C

**Q4.** You go long 1 Nifty future at 24,000 (lot 75). Settlement prices over three days are 24,150, 24,050 and 23,980. What is your cumulative P/L after Day 3?
- A) −₹5,250
- B) −₹7,500
- C) +₹3,750
- D) −₹1,500

> ✅ **Answer: D.** Cumulative P/L = (23,980 − 24,000) × 75 = **−₹1,500**. Option A is only Day 3's P/L ((23,980 − 24,050) × 75), and Option B is Day 2's P/L. 📖 Section 2D

**Q5.** A fund manager holds a ₹27 crore portfolio that tracks the Nifty. Nifty is at 24,000 and the lot size is 75. How many futures lots should she sell for a full hedge?
- A) 112
- B) 15
- C) 150
- D) 1,500

> ✅ **Answer: C.** Contract value = 24,000 × 75 = ₹18 lakh. Lots = 27 cr ÷ 18 lakh = **150**. Option B and Option D are off by a factor of ten. 📖 Section 2F

**Q6.** To roll a long position, a trader sells the near-month Nifty future at 24,060 and buys the next-month future at 24,145 (lot 75). What is the roll cost?
- A) ₹5,250
- B) ₹6,375
- C) ₹85
- D) ₹18,04,500

> ✅ **Answer: B.** Roll cost = (24,145 − 24,060) × 75 = 85 × 75 = **₹6,375**. Option C forgets the lot size, and Option A is the slide's example with different prices. 📖 Section 2E

**Q7.** A company has a ₹50 crore loan at MIBOR + 1.5%. It enters a swap to pay 6.5% fixed and receive MIBOR. What is its effective annual borrowing cost?
- A) 6.5% + MIBOR
- B) MIBOR + 1.5%, because swaps do not change loan cost
- C) 6.5%, because the swap is fixed
- D) 8%, i.e. ₹4 crore, whatever MIBOR does

> ✅ **Answer: D.** MIBOR paid on the loan is offset by MIBOR received on the swap, leaving 6.5% fixed + 1.5% spread = **8%** = ₹4 crore on ₹50 crore. Option C forgets the loan spread. 📖 Section 3

**Q8.** Nifty spot is 24,000. The 24,300 put trades at ₹360. What is its moneyness and time value?
- A) In-the-money; time value ₹60
- B) At-the-money; time value ₹360
- C) In-the-money; time value ₹300
- D) Out-of-the-money; time value ₹360

> ✅ **Answer: A.** For a put, K > S means ITM. Intrinsic = 24,300 − 24,000 = 300, so time value = 360 − 300 = **₹60**. Option C confuses intrinsic value with time value. 📖 Section 4D

**Q9.** You buy a call with strike ₹500 for a premium of ₹18. The stock closes at ₹530 at expiry. What is your profit per share?
- A) ₹18
- B) −₹18
- C) ₹30
- D) ₹12

> ✅ **Answer: D.** Payoff = max(530 − 500, 0) = 30; profit = 30 − 18 = **₹12**. Option C ignores the premium, and Option B would apply only if the call expired worthless. 📖 Section 4B

**Q10.** You write (sell) 1 lot of a Nifty 24,500 call for a premium of 120 (lot 75). Nifty expires at 24,800. What is your P/L?
- A) −₹9,000
- B) −₹13,500
- C) +₹9,000
- D) −₹22,500

> ✅ **Answer: B.** Writer P/L per unit = 120 − (24,800 − 24,500) = −180; × 75 = **−₹13,500**. Option D ignores the premium received, and Option C assumes the call expired worthless. 📖 Section 4C

**Q11.** You buy a put with strike ₹800 for ₹30. What are its break-even and maximum profit per share?
- A) Break-even ₹800; max profit ₹800
- B) Break-even ₹770; max profit ₹770
- C) Break-even ₹770; max profit unlimited
- D) Break-even ₹830; max profit unlimited

> ✅ **Answer: B.** Break-even = K − premium = 770. The stock can fall at most to zero, so max profit = 800 − 0 − 30 = **₹770**. Option D uses the call break-even, and Option C wrongly treats put profit as unlimited. 📖 Section 7

**Q12.** A trader sells a put with strike ₹600 for a premium of ₹25. What is the maximum loss per share?
- A) ₹625
- B) ₹575
- C) Unlimited
- D) ₹25

> ✅ **Answer: B.** Max loss occurs if the stock goes to zero: 600 − 25 = **₹575** (K − premium). Option D is the maximum profit, and Option C applies to a naked short call. 📖 Section 7

**Q13.** An ATM call is priced at 180 with delta 0.55 and theta −5 per day. Over 3 days Nifty rises 80 points, and volatility is unchanged. What is the approximate new premium?
- A) 209
- B) 239
- C) 165
- D) 224

> ✅ **Answer: A.** 180 + 0.55 × 80 − 5 × 3 = 180 + 44 − 15 = **209**. Option D ignores theta decay, and Option C counts only the theta. 📖 Section 4F

**Q14.** Which Greek is positive for both a long call and a long put, and largest for at-the-money options near expiry?
- A) Gamma
- B) Theta
- C) Delta
- D) Rho

> ✅ **Answer: A.** Gamma is positive for all long options and peaks ATM near expiry, when delta flips quickly. Option C has opposite signs for calls and puts, and Option B is negative for long options. 📖 Section 5E

**Q15.** A European call on an index costs 260. S·e^(−qT) = 5,000 and K·e^(−rT) = 4,900. By put-call parity, what should the matching put cost?
- A) 160
- B) 100
- C) 260
- D) 360

> ✅ **Answer: A.** C − P = S·e^(−qT) − K·e^(−rT) = 100, so P = 260 − 100 = **160**. Option D adds the difference instead of subtracting it. 📖 Section 5C

**Q16.** Which input does NOT appear in the Black-Scholes formula?
- A) The risk-free rate
- B) Time to expiry
- C) The expected return of the underlying
- D) Volatility

> ✅ **Answer: C.** Because the option can be replicated and hedged, pricing is risk-neutral: the index is assumed to grow at r − q, so the expected return drops out. Only volatility, rates, dividends, time, spot and strike matter. 📖 Section 5A

**Q17.** In Black-Scholes, what does N(d2) represent for a call?
- A) The option's time value
- B) The expected return of the index
- C) The risk-neutral probability that the option finishes in-the-money
- D) The option's delta

> ✅ **Answer: C.** N(d2) is the risk-neutral chance that S > K at expiry. Option D is N(d1) times e^(−qT), not N(d2). 📖 Section 5C

**Q18.** At 16% volatility, Black-Scholes prices a call at 274. The same call trades in the market at 330. What can you conclude?
- A) The risk-free rate must be higher than assumed
- B) The implied volatility is lower than 16%
- C) The option is mispriced and should be sold immediately
- D) The implied volatility is higher than 16%

> ✅ **Answer: D.** Premium rises with volatility (positive vega), so a higher market price implies a higher σ (≈ 19.5% in the slides). Option C is wrong because traders quote options in implied volatility; a different price usually means a different volatility, not a free profit. 📖 Section 5H

**Q19.** An investor holds a Nifty portfolio worth 24,000 × 75 and buys a 23,600 put for 120. What is the maximum loss, however far the market falls?
- A) ₹9,000
- B) ₹30,000
- C) ₹39,000
- D) Unlimited

> ✅ **Answer: C.** Max loss = (24,000 − 23,600 + 120) × 75 = 520 × 75 = **₹39,000**. Option A is just the premium, and Option B ignores the premium. 📖 Section 8A

**Q20.** An exporter will receive USD 1 million and buys a USD put with strike ₹87.50 for a premium of ₹0.50. What is the guaranteed minimum rupee revenue (floor)?
- A) ₹8.70 crore
- B) ₹8.80 crore
- C) ₹8.95 crore
- D) ₹8.75 crore

> ✅ **Answer: A.** Floor = (87.50 − 0.50) × 1,000,000 = **₹8.70 crore**. Option D ignores the premium, and Option C is the net revenue if spot ends at ₹90. 📖 Section 8C

**Q21.** An investor buys shares at ₹900 and sells a call with strike ₹950 for ₹20. What are the maximum profit and break-even per share?
- A) Unlimited profit; break-even ₹880
- B) Max profit ₹20; break-even ₹900
- C) Max profit ₹50; break-even ₹920
- D) Max profit ₹70; break-even ₹880

> ✅ **Answer: D.** Max profit = (950 − 900) + 20 = **₹70**; break-even = 900 − 20 = **₹880**. Option C forgets the premium, and Option A ignores the capped upside of a covered call. 📖 Section 9A

**Q22.** A trader shorts a stock at ₹1,000 and sells a put with strike ₹950 for ₹15. What are the maximum profit and break-even?
- A) Max profit ₹65; break-even ₹985
- B) Max profit ₹65; break-even ₹1,015
- C) Max profit ₹15; break-even ₹985
- D) Max profit ₹50; break-even ₹1,015

> ✅ **Answer: B.** Max profit = (1,000 − 950) + 15 = **₹65**; break-even = 1,000 + 15 = **₹1,015** (above that the short stock loses). Option A subtracts the premium as if it were a covered call. 📖 Section 9B

**Q23.** An exporter sells USD 500,000 forward at ₹88. The export order is then cancelled, and the rupee weakens to ₹90 at maturity. What happens?
- A) The forward gains ₹10 lakh
- B) The forward loses ₹1 lakh
- C) The forward loses ₹10 lakh with no receivable to offset it
- D) There is no effect, because the hedge was for a cancelled order

> ✅ **Answer: C.** The exporter must deliver USD bought at 90 against a sale at 88: (90 − 88) × 500,000 = **₹10 lakh** loss. Without the receivable, the hedge has become a speculation (over-hedging). Option B drops a zero. 📖 Section 8B

**Q24.** A dealer has sold 20 index calls with delta 0.40 and a multiplier of 100. How should the dealer delta-hedge?
- A) Buy 8 index units
- B) Buy 800 index units
- C) Buy 2,000 index units
- D) Sell 800 index units

> ✅ **Answer: B.** Short calls have negative delta: −20 × 100 × 0.40 = −800 units, so **buy 800** units to neutralise it. Option C ignores delta, and Option D doubles the exposure. 📖 Section 5G

**Q25.** One Nifty lot is 24,000 × 75 and the exchange margin is 12%. Nifty moves 2% against you. What fraction of your margin is lost?
- A) About 6%
- B) 2%
- C) About 16.7%
- D) 12%

> ✅ **Answer: C.** Exposure = ₹18 lakh; margin = ₹2.16 lakh. A 2% move = ₹36,000, which is 36,000 ÷ 2,16,000 ≈ **16.7%** of margin. Leverage magnifies the move by 1/0.12 ≈ 8.3×. Option B ignores leverage. 📖 Section 1C
