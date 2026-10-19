# Session 2: Financial Statements (Balance Sheet, Income Statement, Cash Flow)

> 🎓 In the last class we learned the language of accounting. Today we read the three "books" written in that language: the **Balance Sheet, Income Statement and Cash Flow Statement**. Every investor, banker and analyst reads these three.

**By the end of this session you will be able to:**
- Explain the purpose and role of financial statements
- Identify the key elements of the Balance Sheet (assets, liabilities, equity)
- Calculate depreciation (SLM and RBM) and the treatment of intangible assets
- Build the structure of an Income Statement (from Revenue to EPS)
- Classify cash flows into Operating, Investing and Financing
- Read simple statements and judge a business's performance, position and cash

> 🔁 **Session 1 recap (one line):** `Assets = Liabilities + Equity`, and `Equity = Assets − Liabilities` (the owner's residual claim).

---

## 1. The Three Financial Statements: Big Picture

| Statement | What it tells you | Time frame | In one line |
|---|---|---|---|
| **Balance Sheet** (Statement of Financial Position) | What you **own**, what you **owe**, and what belongs to the owner | **A specific date** (e.g. 31 March 2025) | A **photo** of the business 📸 |
| **Income Statement** (P&L) | Whether there was a profit or a loss | **A period** (month/quarter/year) | A **movie** of the business 🎬 |
| **Cash Flow Statement** | Where cash came from and where it went | **A period** | The business's **bank passbook** 🏦 |

> 💡 **In simple words:** Balance sheet "as at 31 March" bolti hai (ek din ki photo). P&L aur Cash Flow "for the year ended 31 March" bolte hain (poore saal ki kahani).

---

## 2. Balance Sheet (Statement of Financial Position)

It answers 3 questions:
- **A.** What does the business own? → **Assets**
- **B.** What does the business owe? → **Liabilities**
- **C.** What is left for the owners after paying liabilities? → **Equity**

### Ind AS Balance Sheet layout (summary of the ABC Ltd proforma)

| ASSETS | EQUITY & LIABILITIES |
|---|---|
| **I. Non-Current Assets:** PPE (land, building, machinery, vehicles, furniture, CWIP), Intangibles (goodwill, patents, software, brands, licences), LT Investments (subsidiaries, associates, JVs), Deferred tax assets | **I. Equity:** Equity share capital, Other equity (securities premium, retained earnings, general/capital reserve) |
| **II. Current Assets:** Inventories (raw material, WIP, finished goods), Current investments, Trade receivables, Cash & cash equivalents, Prepaid expenses, Advances to suppliers/employees | **II. Non-Current Liabilities:** LT borrowings, lease liabilities, provisions (employee benefits), deferred tax liabilities |
| | **III. Current Liabilities:** ST borrowings, trade payables, advances from customers, statutory dues, unearned revenue, short-term provisions |
| **TOTAL ASSETS** | **= TOTAL EQUITY + LIABILITIES** |

> 💡 In India the balance sheet follows **Ind AS** (Indian Accounting Standards) and the **Schedule III** (Companies Act 2013) format. Both sides always total the same.

> 💡 **CWIP (Capital Work-in-Progress)** = a factory/plant still under construction and not yet in use, so no depreciation is charged on it yet.

---

## 3. Non-Current Assets

**Definition:** Assets the business will hold or use for **more than one accounting period (normally >12 months)**. Their job is to **support the business over the long term**, not to be sold right away.

**Examples:** PPE (factory building, machinery, vehicles, furniture & fittings), long-term investments, intangible assets, right-of-use (lease) assets.

### 3A. Depreciation 📉

**Depreciation** = spreading the cost of PPE over its **useful life**, i.e. the years in which the asset helps earn revenue.

**Why?** A machine will work for 5 years, so charging its full cost to the first year's P&L would be wrong. Depreciation follows the **matching principle**: each year's revenue is matched with that year's cost of "using the machine".

| Term | Meaning | Example |
|---|---|---|
| **Cost** | Initial recognised amount of the asset | Machine ₹10,00,000 |
| **Residual Value** | Expected amount from selling it at the end of its life (net of disposal cost) | ₹1,00,000 after 5 years |
| **Useful Life** | How many years it will be used, **or** how many units it will produce | 5 years |
| **Depreciable Amount** | Cost − Residual Value (the amount that will be "used up") | ₹11,00,000 − ₹1,00,000 = ₹10,00,000 |

> ⚠️ The slide's Depreciable Amount example uses a cost of ₹11,00,000, while the other examples use ₹10,00,000. The formula is the same; only the numbers differ.

### 3B. Straight Line Method (SLM): same depreciation every year

$$\text{Annual Depreciation} = \frac{\text{Cost − Residual Value}}{\text{Useful Life}}$$

**Example:** Machine cost ₹10,00,000, residual ₹1,00,000, life 5 years.
Annual Depreciation = (10,00,000 − 1,00,000) / 5 = **₹1,80,000 per year**

| Year | Opening | Depreciation | Closing |
|---|---|---|---|
| 1 | 10,00,000 | 1,80,000 | 8,20,000 |
| 2 | 8,20,000 | 1,80,000 | 6,40,000 |
| 3 | 6,40,000 | 1,80,000 | 4,60,000 |
| 4 | 4,60,000 | 1,80,000 | 2,80,000 |
| 5 | 2,80,000 | 1,80,000 | **1,00,000** ✅ |

✔️ Check: 5 × 1,80,000 = 9,00,000 total depreciation. Closing value = residual value ₹1,00,000 ✅

### 3C. Reducing Balance Method (RBM / WDV): a fixed % of the opening value each year

$$\text{Annual Depreciation} = \text{Opening Balance} \times \text{Depreciation Rate}$$

**Example:** Truck cost ₹20,00,000, rate 25%.

| Year | Opening | Depreciation @25% | Closing |
|---|---|---|---|
| 1 | 20,00,000 | 5,00,000 | 15,00,000 |
| 2 | 15,00,000 | 3,75,000 | 11,25,000 |
| 3 | 11,25,000 | 2,81,250 | 8,43,750 |
| 4 | 8,43,750 | 2,10,938 | 6,32,812 |
| 5 | 6,32,812 | 1,58,203 | 4,74,609 |

> 💡 **In simple words:** RBM mein **shuru ke saalon mein zyada** depreciation lagta hai, baad mein kam. Trucks/computers jaise assets jo shuru mein tezi se value khote hain, unke liye ye realistic hai. Value **kabhi exactly zero nahi hoti**.

> ⚠️ The slide says useful life 5 years and residual NIL, but the table only goes to 4 years. Even after year 5, about ₹4.75L remains, so a 25% rate doesn't match "NIL residual in 5 years". The slide also calls the truck a "machine".

| | SLM | RBM |
|---|---|---|
| Depreciation pattern | Same every year | Higher first, lower later |
| Base | Cost − Residual | Opening carrying amount |
| Best for | Building, furniture (uniform use) | Vehicles, tech (early obsolescence) |

### 3D. Long-term Investments

Held for more than one year, for two reasons:

| Type | Purpose | Example |
|---|---|---|
| **Strategic** | To support/strengthen the business strategy | Stake in a subsidiary, JV, associate |
| **Financial** | Only to earn a return (dividend, interest, capital appreciation) | Bonds, portfolio shares |

**Real examples:**

| Company | Strategic | Financial | Total |
|---|---|---|---|
| **Tata Motors** (standalone, 31 Mar 2025) | ₹29,718 Cr (subsidiaries, JVs, associates) | ₹1,709 Cr (other investments) | ₹31,427 Cr |
| **Volkswagen** (consolidated, 31 Dec 2025) | €10.0 bn equity-accounted (associates/JVs) | €11.7 bn other financial assets, plus €7.4 bn other equity investments (mostly financial) | €29.1 bn |

- **Tata Motors strategic examples:** TM Passenger Vehicles, TML Commercial Vehicles, TML Smart City Mobility, Digital.AI Labs (subsidiaries), Tata Motors Global Services (JV), Tata Technologies, Automobile Corp of Goa and Tata Hitachi (associates). In FY25 it had 93 subsidiaries, 10 associates and 4 JVs.
- **Volkswagen strategic stakes:** QuantumScape 16% (solid-state battery), Gotion 24% (EV battery supply chain), Sinotruk 25%, FAW-VW 40%, SAIC-VW 50%, SAIC-VW Sales 48%. These are all for the China market and battery tech.
- **VW financial (€11.692 bn):** loans/bonds receivables €6.931 bn + misc €1.059 bn + derivatives €3.702 bn.

> 💡 **In simple words:** Strategic investment = "partner/control ke liye kharida". Financial investment = "paisa banane ke liye kharida". Same share dono ho sakta hai, **intention** decide karta hai.

### 3E. Intangible Assets

**Definition:** **Identifiable, non-monetary** assets with **no physical form**, controlled by the business, and expected to give future benefits.

**Examples:** Patents, copyrights, trademarks, brands, software licences, franchises, customer relationships, technology, R&D (under certain conditions).
**Goodwill** is treated separately: it arises **only in a merger/acquisition**, when the acquirer pays more than the fair value of net assets.

**Recognition: both conditions are required:**
1. **Probable future economic benefits** must flow to the entity
2. **Cost must be reliably measurable**

| Type | Initial value | Example |
|---|---|---|
| **Purchased / externally acquired** | Cost | Patent/software/licence bought for ₹10 Cr |
| **Internally generated** | Cost (only eligible development cost) | Company built its own software. The research phase is expensed; the development phase is capitalised once the criteria are met |

> ⚠️ On the slide, the explanation for "Reliable measurement of cost" is a copy of the previous line by mistake. The internally generated example is also a copy of "Purchased patent...". A correct example: in-house developed software/product design.

> 💡 An internally built **goodwill/brand** is never recognised, because its cost cannot be measured reliably.

**Subsequent measurement: 2 models**

| Model | Carrying Amount |
|---|---|
| **Cost model** | Cost − Accumulated **amortisation** − Accumulated impairment losses |
| **Revaluation model** | Fair value (at revaluation date) − later accumulated amortisation/impairment |

> 💡 **In simple words:** **Amortisation** = intangible ka depreciation. **Impairment** = value ka achaanak girna (e.g. patent obsolete ho gaya).

**Example:** Patent ₹10 Cr, life 5 years → amortisation ₹2 Cr/yr. After 2 years: 10 − 4 = ₹6 Cr. Then a ₹1 Cr impairment → carrying amount **₹5 Cr**.

---

## 4. Liabilities

| Non-Current (settled after 12 months) | Current (settled within 12 months / operating cycle) |
|---|---|
| Long-term bank loans | Trade payables |
| Debentures & bonds payable | Short-term borrowings |
| Long-term lease liabilities | Accrued expenses (salary, wages, interest payable) |
| Long-term provisions | Tax payable |
| Deferred tax liabilities | **Current portion of long-term debt** |
| Other long-term borrowings | Contract liabilities / revenue received in advance |
| | Current lease liabilities |

> 💡 **In simple words (current portion of LT debt):** ₹50L ka 5-saal loan hai, jisme se ₹10L agle 12 mahine mein chukaane hain. To ₹10L current liability aur ₹40L non-current. Ek hi loan, do jagah!

> 💡 An **advance from a customer** is a liability, because you still have to deliver the goods/service (same logic as the Netflix example in Session 1).

---

## 5. Equity

$$\text{Equity = Assets − Liabilities}$$

| Component | Meaning |
|---|---|
| **Share capital** | Money received by issuing shares, at **face/par value**. Can be ordinary (equity) and preference shares |
| **Securities premium** | Extra money received **above** face value |
| **Retained earnings** | Accumulated profit not distributed as dividends |

**Example:** A company issued 1,00,000 shares of **₹10 face value** at **₹50**.
- Share capital = 1,00,000 × 10 = **₹10,00,000**
- Securities premium = 1,00,000 × 40 = **₹40,00,000**
- Total cash = ₹50,00,000 ✅

### Mini Balance Sheet: XYZ Ltd (as at 31 March 2026, ₹ lakh)

| Assets | ₹ | Equity & Liabilities | ₹ |
|---|---|---|---|
| PPE (net of depreciation) | 40 | Share capital | 20 |
| Intangibles | 5 | Securities premium | 5 |
| LT investments | 10 | Retained earnings | 18 |
| **Non-current total** | **55** | **Equity total** | **43** |
| Inventory | 8 | LT loan | 20 |
| Trade receivables | 6 | Deferred tax liability | 2 |
| Cash | 4 | **Non-current liab. total** | **22** |
| **Current total** | **18** | Trade payables | 5 |
| | | Current portion of LT debt | 3 |
| | | **Current liab. total** | **8** |
| **TOTAL ASSETS** | **73** | **TOTAL E + L** | **73** ✅ |

---

## 6. Income Statement (Statement of Profit & Loss)

It shows the **financial performance** over a **period**.
**Objectives:** determine profitability, measure operating performance, compare performance (year-on-year or against competitors).

### Line items: top to bottom (the "waterfall") 🌊

| # | Line Item | Formula / Meaning |
|---|---|---|
| 1 | **Revenue / Sales** | Earnings from selling products/services. Always the first line |
| 2 | **COGS** | Direct cost: materials, direct labour, production costs |
| 3 | **Gross Profit** | `Revenue − COGS` |
| 4 | **Operating Expenses** | Salaries, rent, advertising, depreciation, insurance, utilities |
| 5 | **Operating Profit (EBIT)** | `Gross Profit − Operating Expenses` (profit from the core business) |
| 6 | **Other Income** | Outside the main business: interest, dividends, profit on asset sale |
| 7 | **Other Expenses / Finance Costs** | Loan interest, finance charges, loss on asset sale |
| 8 | **PBT** | `Operating Profit + Other Income − Other Expenses` |
| 9 | **Income Tax** | Tax on taxable income |
| 10 | **PAT / Net Profit** | `PBT − Tax` (the bottom line) |
| 11 | **EPS** | `Net Profit ÷ Number of outstanding shares` |

> 💡 **In simple words:** **EBIT** = Earnings Before Interest & Tax. Ye batata hai business **operationally** kitna strong hai, loan structure aur tax hata ke. Do companies compare karne ke liye best.

### Worked Example: FreshMart Ltd (FY 2025-26)

| Particulars | ₹ |
|---|---|
| Revenue | 80,00,000 |
| Less: COGS | (48,00,000) |
| **Gross Profit** | **32,00,000** |
| Less: Salaries 8L + Rent 4L + Depreciation 3L + Advertising 2L | (17,00,000) |
| **Operating Profit (EBIT)** | **15,00,000** |
| Add: Interest income | 1,00,000 |
| Less: Finance costs (loan interest) | (2,00,000) |
| **PBT** | **14,00,000** |
| Less: Tax @25% | (3,50,000) |
| **PAT** | **10,50,000** |
| Shares outstanding | 5,00,000 |
| **EPS** | 10,50,000 ÷ 5,00,000 = **₹2.10** |

✔️ Check: 80 − 48 = 32; 32 − 17 = 15; 15 + 1 − 2 = 14; 14 × 25% = 3.5; 14 − 3.5 = 10.5 ✅

> ⚠️ In the slide's Ind AS proforma, other income is added first into "Total Income (I+II)" and COGS is subtracted after that. In the class formula, other income comes **after** EBIT. The final PAT is the same either way; only the Gross Profit figure looks different. Use the class order in the exam.

---

## 7. Cash Flow Statement

Shows the inflows and outflows of **cash and cash equivalents** during the period.

> 💡 **In simple words:** **Income Statement** batata hai profit kitna hua. **Cash Flow** batata hai **cash position kaise badli**. Profit ≠ Cash, kyunki accrual accounting mein credit sales profit mein aa jaati hain par cash abhi aaya hi nahi!

**Three activities:**

| Activity | What it covers | Inflow examples | Outflow examples |
|---|---|---|---|
| **Operating (CFO)** | Day-to-day business | Cash from customers, cash for services | Suppliers, salaries, rent, taxes |
| **Investing (CFI)** | Long-term assets & investments | **Selling** machinery/buildings/investments, interest/dividend received | **Buying** machinery/buildings/investments |
| **Financing (CFF)** | Dealings with owners and lenders | Issuing shares/debentures, taking loans | Buyback, dividend paid, interest paid, loan repayment |

$$\text{Closing Cash = Opening Cash + CFO + CFI + CFF}$$

### 7A. CFO: Indirect Method (Net Profit → Cash)

Net profit is on an accrual basis, so you need **2 types of adjustments** to get to cash:

**1. Non-cash items (add back):**
- **Depreciation & Amortisation:** reduce profit, but no cash leaves
- **Stock-based compensation:** shares given to employees, no cash paid
- **Deferred taxes:** difference between book tax and tax actually paid
- **Subtract** a gain on asset sale and **add** a loss. This is an investing activity, and its full cash shows in CFI

**2. Working capital changes:**

| Item | Increase ⬆️ | Decrease ⬇️ | Logic |
|---|---|---|---|
| **Trade Receivables** | Subtract | Add | Sale recorded but cash not received / cash received for an old sale |
| **Inventory** | Subtract | Add | Cash spent on buying stock / old stock sold |
| **Trade Payables** | Add | Subtract | Bill kept pending, cash saved / supplier paid off |

> 💡 **Trick to remember:** Current **asset** goes up → cash goes down (−). Current **liability** goes up → cash is saved (+).

> ⚠️ On the slide, the label ("Accounts Receivable:") of the first working capital bullet is missing. "Subtract increases (sales recorded, cash uncollected)" refers to receivables.

### Worked Example: Calculate CFO

Net profit ₹10L, depreciation ₹3L, receivables ⬆️ ₹2L, inventory ⬇️ ₹1L, payables ⬆️ ₹1.5L.

| Item | ₹ lakh |
|---|---|
| Net profit | 10.0 |
| + Depreciation | +3.0 |
| − Increase in receivables | −2.0 |
| + Decrease in inventory | +1.0 |
| + Increase in payables | +1.5 |
| **CFO** | **13.5** |

✔️ 10 + 3 − 2 + 1 + 1.5 = 13.5 ✅

> 🎯 **Analyst tip:** If CFO is consistently **lower than PAT**, that's a warning sign. Profit is only "on paper" and cash isn't coming in (receivables are piling up).

---

## 8. Quick Revision 🧠

- **Balance Sheet** = photo on one date (A = L + E). **P&L** = profit for a period. **Cash Flow** = cash movement for a period
- **Non-current assets:** PPE, LT investments (strategic vs financial), intangibles, ROU assets
- **SLM:** `(Cost − RV) / Life`, same every year. **RBM:** `Opening × Rate`, higher at first
- **Intangibles:** recognise when benefit is probable and cost is reliable. Cost model or revaluation model. Goodwill only in M&A
- **Equity** = Share capital + Securities premium + Retained earnings
- **P&L waterfall:** Revenue → GP → EBIT → PBT → PAT → EPS
- **CFO (indirect):** Net profit + non-cash items ± working capital changes
- **CFI** = buying/selling long-term assets. **CFF** = shares, loans, dividends, interest paid

---

## 9. MCQs (Mid–Advanced Level) 📝

**Q1.** A machine costs ₹12,00,000, with a residual value of ₹2,00,000 and a useful life of 4 years. What is its carrying amount after 3 years under SLM?
- A) ₹2,50,000
- B) ₹5,00,000
- C) ₹3,00,000
- D) ₹4,50,000

> ✅ **Answer: D.** Annual depreciation = (12 − 2)/4 = ₹2.5L. 3 years = ₹7.5L. 12 − 7.5 = **₹4.5L**. 📖 Section 3B

**Q2.** A van costing ₹8,00,000 is depreciated at 20% under RBM. What is the closing balance at the end of Year 2?
- A) ₹6,40,000
- B) ₹5,12,000
- C) ₹4,96,000
- D) ₹4,80,000

> ✅ **Answer: B.** Y1: 8L × 20% = 1.6L → 6.4L. Y2: 6.4L × 20% = 1.28L → **5.12L**. Option D applies 20% to cost in both years (as in SLM). 📖 Section 3C

**Q3.** A machine costs ₹15,00,000 and has an expected residual value of ₹1,50,000. What is the depreciable amount?
- A) ₹16,50,000
- B) ₹1,50,000
- C) ₹15,00,000
- D) ₹13,50,000

> ✅ **Answer: D.** Cost − Residual = 15 − 1.5 = **₹13.5L**. 📖 Section 3A

**Q4.** For the same cost and life, what is the effect of RBM (at a high rate) on **first-year profit** compared with SLM?
- A) No difference
- B) Profit will be lower
- C) Profit will be higher
- D) Cash flow will fall

> ✅ **Answer: B.** RBM charges more depreciation in the early years, so profit is lower. There is no effect on cash, because depreciation is non-cash, which is why D is wrong. 📖 Section 3C

**Q5.** Software is bought for ₹6 Cr: life 4 years, nil residual, cost model. An impairment of ₹0.5 Cr occurs in Year 2. What is the carrying amount at the end of Year 2?
- A) ₹3.5 Cr
- B) ₹2.5 Cr
- C) ₹4.0 Cr
- D) ₹3.0 Cr

> ✅ **Answer: B.** Amortisation = 6/4 = 1.5 Cr/yr. 2 years = 3 Cr. 6 − 3 − 0.5 = **₹2.5 Cr**. 📖 Section 3E

**Q6.** A company issues 2,00,000 shares of ₹10 face value at ₹35. What are the share capital and securities premium?
- A) SC ₹20L; SP ₹50L
- B) SC ₹20L; SP ₹70L
- C) SC ₹70L; SP ₹0
- D) SC ₹50L; SP ₹20L

> ✅ **Answer: A.** SC = 2L × 10 = ₹20L. SP = 2L × (35 − 10) = **₹50L**. Total cash = ₹70L ✅. 📖 Section 5

**Q7.** Revenue ₹60L, COGS ₹36L, operating expenses ₹12L, other income ₹2L, finance costs ₹3L, tax 25%. What is PAT?
- A) ₹8.25L
- B) ₹6.75L
- C) ₹9.00L
- D) ₹11.00L

> ✅ **Answer: A.** GP = 24; EBIT = 12; PBT = 12 + 2 − 3 = 11; Tax = 2.75; PAT = **₹8.25L**. 📖 Section 6

**Q8.** The company in Q7 has 3,00,000 shares outstanding. What is EPS?
- A) ₹2.75
- B) ₹3.00
- C) ₹3.67
- D) ₹2.25

> ✅ **Answer: A.** 8,25,000 ÷ 3,00,000 = **₹2.75**. 📖 Section 6

**Q9.** A company increases its loan, raising interest cost by ₹5L. Everything else is unchanged. Which figure **stays the same**?
- A) PBT
- B) EPS
- C) Operating Profit (EBIT)
- D) PAT

> ✅ **Answer: C.** Interest is a finance cost that sits **below** EBIT. So EBIT is unchanged, while PBT, PAT and EPS all fall. 📖 Section 6

**Q10.** Net profit ₹20L, depreciation ₹5L, receivables ⬆️ ₹4L, inventory ⬆️ ₹3L, payables ⬇️ ₹2L. What is CFO?
- A) ₹16L
- B) ₹26L
- C) ₹20L
- D) ₹34L

> ✅ **Answer: A.** 20 + 5 − 4 − 3 − 2 = **₹16L**. Higher receivables/inventory reduce cash; lower payables mean cash went out. 📖 Section 7A

**Q11.** Net profit of ₹10L includes a ₹1L gain on sale of a machine. Depreciation is ₹2L. What is CFO before working capital changes?
- A) ₹11L
- B) ₹9L
- C) ₹13L
- D) ₹12L

> ✅ **Answer: A.** 10 + 2 (add back depreciation) − 1 (remove the gain, because the machine sale cash goes to CFI) = **₹11L**. 📖 Section 7A

**Q12.** In which activity does a dividend paid to shareholders appear?
- A) It does not appear in the cash flow at all
- B) Financing
- C) Investing
- D) Operating

> ✅ **Answer: B.** It is a transaction with owners, so it is a **financing outflow**. 📖 Section 7

**Q13.** A manufacturing company **receives** interest on a fixed deposit. Per the class proforma (Ind AS 7), where does this appear?
- A) Operating
- B) Investing
- C) Financing
- D) Other comprehensive income

> ✅ **Answer: B.** In the proforma, interest received and dividends received are under **investing**. Interest **paid** goes to financing. 📖 Section 7

**Q14.** A patent is bought for ₹10 Cr in cash. Where does this appear in the cash flow statement?
- A) Operating outflow
- B) Nowhere, because it is intangible
- C) Investing outflow
- D) Financing outflow

> ✅ **Answer: C.** A patent is a long-term (intangible) asset, and buying a long-term asset is an **investing** activity. 📖 Sections 3E, 7

**Q15.** Under the CFO indirect method, how is **stock-based compensation** treated?
- A) Shown under financing
- B) Added back
- C) Ignored
- D) Subtracted

> ✅ **Answer: B.** This expense reduced profit but no cash left the business, so it is added back. 📖 Section 7A

**Q16.** A company has a ₹50L term loan, of which ₹10L is repayable in the next 12 months. How is it shown on the balance sheet?
- A) ₹50L non-current
- B) ₹10L non-current + ₹40L current
- C) ₹40L non-current + ₹10L current
- D) ₹50L current

> ✅ **Answer: C.** The part due in the next 12 months is the **current portion of long-term debt**; the rest is non-current. 📖 Section 4

**Q17.** A SaaS company receives a 1-year subscription in advance from a customer. The service has not started yet. What is this on the balance sheet?
- A) Current asset
- B) Revenue
- C) Current liability (contract liability)
- D) Equity

> ✅ **Answer: C.** The service is still owed, which is an obligation, so it is a liability. Revenue is recognised as the service is delivered. 📖 Section 4

**Q18.** A brand the company built itself over many years has an estimated value of ₹500 Cr. Can it be shown on the balance sheet as an intangible asset?
- A) Yes, at ₹500 Cr
- B) Only under the name of goodwill
- C) Yes, using the revaluation model
- D) No, because the cost of an internally generated brand cannot be measured reliably

> ✅ **Answer: D.** The "reliable measurement of cost" recognition condition fails. Brands/goodwill are recognised only when **acquired**. 📖 Section 3E

**Q19.** Which statement is dated "**as at 31 March**" rather than "for the year ended"?
- A) Cash Flow Statement
- B) Income Statement
- C) All three
- D) Balance Sheet

> ✅ **Answer: D.** The balance sheet is a photo at a **point in time**. The other two describe a period. 📖 Section 1

**Q20.** A fast-growing company has PAT of ₹50 Cr but **negative** CFO. What is the most likely reason?
- A) A large increase in receivables and inventory
- B) Dividend payment
- C) Depreciation is very high
- D) A large increase in payables

> ✅ **Answer: A.** Cash gets stuck in growing receivables/inventory. C is wrong because depreciation is added back and **increases** CFO. D also increases CFO. B is a financing activity. 📖 Section 7A

**Q21.** A car maker buys a 24% stake in a battery tech company to secure its EV supply chain. What kind of investment is this?
- A) Inventory
- B) Financial (current)
- C) Strategic (non-current)
- D) Cash equivalent

> ✅ **Answer: C.** The stake serves the business strategy (like VW–Gotion), so it is a strategic, long-term investment. 📖 Section 3D

**Q22.** CFO ₹16L, CFI −₹25L, CFF +₹12L, opening cash ₹5L. What is the closing cash?
- A) ₹−4L
- B) ₹58L
- C) ₹3L
- D) ₹8L

> ✅ **Answer: D.** Net change = 16 − 25 + 12 = +3. Closing = 5 + 3 = **₹8L**. 📖 Section 7
