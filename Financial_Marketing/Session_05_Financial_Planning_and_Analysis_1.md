# Session 5: Financial Planning & Analysis – 1 (Introduction to Management Accounting)

> 🎓 In earlier sessions we studied financial accounting, which is the report card for **outsiders**. Today we enter the world of **insiders (managers)**: what cost is, how it behaves, and how we use it to make decisions.

**By the end of this session you will be able to:**
- Explain the role of Management Accounting in decision-making
- Classify costs (nature, function, traceability, behaviour, relevance)
- Apply cost concepts to business/engineering scenarios
- Calculate cost per unit, overhead absorption, marginal cost and contribution
- Make capacity-constraint decisions (accept a special order or not)

---

## 1. What is Management Accounting?

| | Financial Accounting | Management (Managerial) Accounting |
|---|---|---|
| For whom | Outsiders (investors, banks, govt) | Insiders (managers) |
| Rules | Follows **GAAP** (Generally Accepted Accounting Principles) | No fixed rulebook; whatever is useful for the decision |
| Info type | Only financial | **Financial + non-financial** (units, hours, defects, customer data) |
| Time | Record of the past | Future-oriented |

**Definition:** Identifying, measuring, analysing and communicating financial and non-financial information to managers for decision-making.

It supports 3 activities:

| Activity | Meaning | Tool |
|---|---|---|
| **Planning** | Estimating costs, revenue, profit, resources | **Budgeting** |
| **Control** | Checking whether we are following the plan | **Actual vs Budget + Variance Analysis** |
| **Decision-making** | Choosing the best option | Make or buy, outsource vs in-house, product selection, **pricing** |

> 💡 Pricing types on the slide: competitive, penetration (enter the market with a low price), cost-plus (cost + margin), full cost, marginal cost pricing, price skimming (high price first, then reduce slowly).

> 📌 The biggest part of management accounting is **Cost Accounting**.

---

## 2. Basic Cost Concepts

**Cost** = **Sacrifice** of resources (money paid) for an objective, such as buying an asset, making a product or providing a service.

| Term | Meaning | Example |
|---|---|---|
| **Expense** | A cost charged **against the revenue of a period** | This year's salary, rent |
| **Actual cost** | Cost that **has already been incurred** | Last month's actual electricity bill |
| **Budgeted cost** | **Estimated** cost (for the future) | Expected raw material spend for next quarter |
| **Cost object** | Anything whose cost we want to measure | Product, service, project, customer, department, channel |
| **Costing** | The process of measuring the cost of cost objects | How much it cost to build one car |

> 💡 **In simple words:** Har expense ek cost hai, par har cost expense nahi. ₹10L ki machine kharidi → ye **cost** hai jo **asset** ban gayi. Har saal ka depreciation **expense** banta hai jab wo revenue ke against match hota hai.
> ⚠️ The slide shows both "Cost ≠ Expense" and "Expense = Cost", which is confusing. The correct meaning is above: expense is a **subset** of cost.

---

## 3. Cost Classification

**Cost classification** = arranging costs into groups based on a characteristic or purpose. **The same cost can be classified in different ways**, depending on the purpose of the analysis.

> **Example: Steel used in a car**
>
> | Basis | Classification |
> |---|---|
> | Nature | Material cost |
> | Function | Production cost |
> | Behaviour | Variable cost |
> | Traceability | Direct cost |
> | Relevance | Relevant for the production decision |

All this matters because **management must know how much it costs to make 1 unit.**

$$\text{Cost per unit} = \frac{\text{Total production cost}}{\text{Units produced}} = \frac{₹50,00,000}{10,000} = ₹500$$

### 3A. By Nature / Element (**what** the cost is)

| Element | Direct | Indirect |
|---|---|---|
| **Material** | Directly linked to the product: a car's **steel body**, wood in furniture, fabric in garments, phone components | Cannot be economically traced to one product: **lubricants** in a car factory |
| **Labour** | Directly involved in making the product: **assembly-line worker**, machine operator | Supports production but cannot be traced to a unit: **factory security guard**, production supervisor |
| **Other Expenses** | Everything other than material and labour (rent, electricity, insurance...) | |

### 3B. By Function (**where** the cost was incurred)

| Function | Meaning | Examples |
|---|---|---|
| **Production / Manufacturing** | Making the product or providing the service | Raw material, production wages, factory electricity, factory depreciation, factory rent |
| **Administration** | General admin & management | Senior management salary, office insurance, office equipment depreciation, head office rent, audit fees, office supplies, legal fees |
| **Selling** | To **generate sales** | Sales commission, advertising, promotion |
| **Distribution** | To **deliver the product to the customer** | Warehousing, transport, delivery charges |

$$\text{Total Cost = Production + Administration + Selling + Distribution}$$

> 💡 Trick: see the word **factory** → production. **Office/head office** → admin. **Convincing the customer** → selling. **Getting it to the customer** → distribution.

### 3C. By Traceability (can it be **directly attached** to one product?)

| Type | Meaning | Examples |
|---|---|---|
| **Direct Cost** | Can be **traced** to a specific cost object | Steel in a particular car, wood in one table, wages of the worker assembling a specific product |
| **Indirect Cost** = **Overheads** | **Cannot** be traced to a specific cost object | Factory rent, factory electricity, factory insurance, supervisor salary |

### 3D. By Behaviour ⭐ (how does the cost behave when activity increases?)

#### (i) Fixed Cost
Stays **constant in total** within a **relevant range** (existing capacity), however much activity changes.
Examples: factory rent, manager salary, insurance, loan payments, admin cost.

> **Key point:** Total is fixed ✅, but **per unit falls** as production increases.

$$\text{Fixed cost per unit} = \frac{\text{Total Fixed Cost}}{\text{Units}}$$

**Example:** A car company's factory rent is ₹10,00,000/month.

| Cars produced | Total Fixed Cost | Fixed Cost per Car |
|---|---|---|
| 1,000 | ₹10,00,000 | ₹1,000 |
| 2,000 | ₹10,00,000 | ₹500 |
| 5,000 | ₹10,00,000 | ₹200 |
| 10,000 | ₹10,00,000 | ₹100 |

> 💡 This is why big companies produce more to lower per-unit cost (**economies of scale**).

#### (ii) Variable Cost
**Total** cost changes in **proportion** to activity, but stays **constant per unit**.
Examples: direct material, per-unit labour, sales commission, per-unit delivery, production electricity.

$$\text{Total Variable Cost = Variable cost per unit × Output}$$

**Example:** Each bottle uses ₹20 of raw material.

| Bottles | VC per Bottle | Total VC |
|---|---|---|
| 1,000 | ₹20 | ₹20,000 |
| 2,000 | ₹20 | ₹40,000 |
| 5,000 | ₹20 | ₹1,00,000 |
| 10,000 | ₹20 | ₹2,00,000 |

> 🧠 **In simple words (ulta yaad rakho):** Fixed cost → total fixed, per unit variable. Variable cost → total variable, per unit fixed.

#### (iii) Semi-Variable Cost
Has **both fixed and variable parts**. The fixed part is incurred even at **zero** activity, and the variable part rises as activity rises.

$$\text{Semi-variable Cost = Fixed part + (Variable rate × Activity)}$$

On the graph, the line starts **not from zero but from the fixed amount (point B)** and goes up.

**Example: Factory electricity bill.** Fixed demand charge ₹50,000/month + ₹8 per machine-hour.

| Machine Hours | Fixed | Variable | Total | Cost per Machine Hour |
|---|---|---|---|---|
| 1,000 | ₹50,000 | ₹8,000 | ₹58,000 | ₹58 |
| 2,000 | ₹50,000 | ₹16,000 | ₹66,000 | ₹33 |
| 5,000 | ₹50,000 | ₹40,000 | ₹90,000 | ₹18 |
| 10,000 | ₹50,000 | ₹80,000 | ₹1,30,000 | ₹13 |

At 5,000 hrs: ₹50,000 + (₹8 × 5,000) = **₹90,000**

> ⚠️ The slide gives **factory rent, manager salary, security guard, depreciation** as semi-variable examples. These are actually **fixed costs**. Real semi-variable examples: **electricity bill** (as in the table above), telephone/mobile bill (rental + per call), machine maintenance, salesperson's salary + commission.

### 3E. By Relevance (does it matter for the decision?)

| Type | Meaning | Use in decision? |
|---|---|---|
| **Relevant cost** | **Future** cost that **differs** between options | ✅ Yes |
| **Irrelevant cost** | Stays the **same** across options | ❌ No |
| **Sunk cost** | **Already incurred** and **cannot be recovered** | ❌ Never |
| **Opportunity cost** | Benefit lost by giving up the **next best option** | ✅ Yes |

**Sunk cost example:** A tech company has spent ₹2 crore building software. Now it learns a competitor's product is better. Options: (A) Launch, (B) Abandon the project.
The ₹2 crore will not come back **in either case**, so it is a **sunk cost** and should not affect the decision. Compare only **future** costs and revenues.

> 💡 **In simple words:** "Itna paisa laga diya, ab chhod nahi sakte" wali soch ko **sunk cost fallacy** kehte hain. Smart manager ye galti nahi karta.

**Opportunity cost example:** A student has 2 options: a ₹16 lakh/year job or a full-time MBA.
- Chooses MBA → opportunity cost = **₹16 lakh salary** given up
- Chooses the job → opportunity cost = **benefit of the MBA** (better managerial jobs in future)

$$\text{Opportunity Cost = Benefit sacrificed from the next best alternative}$$

---

## 4. Cost Unit

**Cost unit** = the unit of product/service in which cost is **calculated, accumulated and expressed**.

| Industry | Cost Unit |
|---|---|
| Automobile | Per car |
| Hotel | Per room-night |
| Airline | Per passenger-kilometre |
| Electricity company | Per kilowatt-hour (kWh) |
| Hospital | Per patient-day |
| Transport company | Per tonne-kilometre |

> 💡 **In simple words:** Cost object = *kiski* cost chahiye (product, dept, customer). Cost unit = *kis unit mein* cost express karein (per car, per kWh).

---

## 5. Product Cost & Absorption of Overheads

### Why do we need per-unit cost?
To value inventory, decide the **selling price**, measure and compare profitability, control costs, and make production/outsourcing decisions.
> Example: The total cost of a product is ₹80/unit. Only if you know this can you decide whether selling at ₹100 or ₹120 makes sense.

### Components of Product Cost (manufacturing)

| Component | Meaning | Example |
|---|---|---|
| **Direct Materials** | Directly traceable to the product | Aluminium in a car |
| **Direct Labour** | Labour directly making the product | Wages of the worker assembling a machine |
| **Production Overheads** | Other production costs that cannot be traced | Factory rent, electricity, insurance, supervisor salary, maintenance, security |

$$\text{Total Production Cost = Direct Material + Direct Labour + Production Overheads}$$

> 💡 Direct Material + Direct Labour (+ direct expenses) is also called **Prime Cost**.

### Motor example
Electric motor: DM ₹4,000 + DL ₹2,000 = **Direct cost ₹6,000/motor**.
But factory rent, electricity, machine depreciation, supervisor salary, maintenance and security are also incurred. These are all **production overheads**.
**Question:** How much of these should be charged to **each motor**? → The answer is **Overhead Absorption**.

> 🧮 *(Illustration; the slide has no numbers)* Suppose the year's budgeted overheads are ₹30,00,000 and 1,000 motors will be made → OAR = ₹3,000/motor → **Full production cost = 6,000 + 3,000 = ₹9,000/motor**.

### Overhead Absorption
Charging a **share of total production overheads to each unit**, using a suitable **absorption rate**.
> 💡 **In simple words:** overheads ko un products mein **baant do** jo unka fayda uthate hain.

$$\textbf{OAR (Overhead Absorption Rate)} = \frac{\text{Budgeted Overhead}}{\text{Budgeted Activity}}$$

**Example:** Factory overhead $100,000, units 20,000 → $100,000 ÷ 20,000 = **$5 per unit**.

> 💡 "Activity" is not only units. It can also be **machine-hours** (machine-heavy factory) or **labour-hours** (labour-heavy factory). Pick the basis that best drives the overheads.
> ⚠️ This example on the slide is in **$ (dollars)**, while everything else is in ₹. The concept is the same.

---

## 6. Marginal Cost & Marginal Costing ⭐

### Marginal Cost (MC)
The **additional cost** of producing **one extra unit**.
In practice it is the **variable cost of one extra unit** (assuming fixed cost does not change).

$$\text{Marginal Cost} = \frac{\text{Change in Total Cost}}{\text{Change in Output}}$$

**Example:** 10,000 components → ₹50,00,000. 10,100 components → ₹50,60,000.
- Output change = 100 units, Cost change = ₹60,000
- **MC = ₹60,000 / 100 = ₹600 per extra component**

### Marginal Costing (technique)
A costing **technique** in which:
- **Variable costs → charged to products**
- **Fixed costs → treated as period costs** (deducted directly from that period's profit, not included in inventory)

> ⚠️ **Marginal Costing ≠ Marginal Cost.** One is a technique, the other is a number. The slide labels both "(MC)", so don't get confused.

**Central idea = Contribution**

$$\text{Contribution = Sales − Variable Cost}$$
$$\text{Contribution per unit = Selling Price − Variable cost per unit = SP − MC}$$
$$\text{Profit = Total Contribution − Fixed Costs}$$

> 💡 **In simple words:** **Contribution** = har unit bechke jo paisa bachta hai **fixed costs cover karne aur profit banane** ke liye. Pehle fixed costs cover hoti hain, phir profit shuru hota hai.

---

## 7. Decision-Making with Marginal Costing ✈️

### Case 1: **No capacity constraint** (spare capacity available)
An airline has one empty seat. Normal fare ₹8,000. Aircraft lease, crew salary and airport charges are already committed (fixed).
Cost of one extra passenger: Meal ₹500 + Service ₹200 + Processing ₹100 + Other ₹200 = **MC ₹1,000**.
A last-minute customer is willing to pay **₹3,000**.

**Extra contribution = 3,000 − 1,000 = ₹2,000** → ✅ **Sell it!** The seat would have gone empty anyway.

> 💡 Rule: With spare capacity, accept any order where **price > marginal cost**. Fixed cost is incurred anyway, so it is irrelevant.

### Case 2: **Capacity constraint** (limited resource)
**Capacity constraint** = demand or required capacity exceeds available capacity. The scarce resource that creates the restriction is called the **limiting factor / bottleneck**.

$$\text{Demand (Required Capacity)} > \text{Available Capacity}$$

**Example: Bengaluru–Mumbai flight**
- 180 seats; 175 already sold at an average of ₹8,000 → **only 5 seats left**
- A travel company wants **10 seats @ ₹3,000** (you can give only 5)
- MC per passenger = ₹1,000
- If not given to the travel company, the seats can be sold to individual passengers at **₹7,000**

| # | Step | Calculation | Answer |
|---|---|---|---|
| 1 | Marginal cost/passenger | 500 + 200 + 100 + 200 | **₹1,000** |
| 2 | Contribution: travel co. | 3,000 − 1,000 | **₹2,000/seat** (5 seats → ₹10,000) |
| 3 | Contribution: individual | 7,000 − 1,000 | **₹6,000/seat** (5 seats → ₹30,000) |
| 4 | Opportunity cost per seat | Contribution given up from individuals = 7,000 − 1,000 | **₹6,000** |
| 5 | Minimum acceptable price | MC + Opportunity cost = 1,000 + 6,000 | **₹7,000** |
| 6 | Accept offer? | 3,000 < 7,000 → loss of 5 × (6,000 − 2,000) | ❌ **Reject**, ₹20,000 less contribution |

$$\textbf{Minimum Acceptable Price (capacity constraint) = Marginal Cost + Opportunity Cost}$$

> 💡 **Case 1 vs Case 2:**
> - Spare capacity → minimum price = **MC** (₹1,000)
> - Full capacity → minimum price = **MC + opportunity cost** (₹7,000)
>
> **In simple words:** Same seat, same cost, par **situation** se decision badal jaata hai. Yahi management accounting ki power hai.

> ⚠️ Step 3 on the slide says "₹6,000 − ₹1,000 = ₹7,000". The correct calculation is **₹7,000 − ₹1,000 = ₹6,000** (later steps also use ₹6,000). The step 6 title "accept airline accept" is also a typo.

---

## 8. Quick Revision 🧠

- **Mgmt accounting** = internal, future-oriented, financial + non-financial info → Planning (budget), Control (variance), Decisions
- **Cost** = sacrifice of resources; **Expense** = cost matched against a period's revenue
- **5 classifications:** Nature (material/labour/other), Function (production/admin/selling/distribution), Traceability (direct/indirect = overheads), Behaviour (fixed/variable/semi-variable), Relevance (relevant/irrelevant/sunk/opportunity)
- **Fixed:** total constant, per unit ⬇️ · **Variable:** total ⬆️, per unit constant · **Semi-variable:** fixed + variable
- **Production cost** = DM + DL + Production overheads
- **OAR** = Budgeted overhead ÷ Budgeted activity
- **MC** = ΔTotal cost ÷ ΔOutput ≈ variable cost per unit
- **Contribution** = Sales − Variable cost; **Profit** = Contribution − Fixed cost
- **Min price:** spare capacity → MC; capacity constraint → MC + Opportunity cost
- Never include **sunk cost** in a decision

---

## 9. MCQs (Mid–Advanced Level) 📝

**Q1.** A factory's monthly rent is ₹6,00,000. Production rises from 2,000 units to 3,000 units (within capacity). What is the effect on fixed cost per unit?
- A) Total rent becomes ₹9,00,000
- B) Rises from ₹200 to ₹300, i.e. ₹100 higher
- C) Falls from ₹300 to ₹200, i.e. ₹100 lower
- D) Stays at ₹300

> ✅ **Answer: C.** 6,00,000 ÷ 2,000 = ₹300 and 6,00,000 ÷ 3,000 = ₹200. The total stays fixed while the per-unit amount falls. 📖 Section 3D

**Q2.** Raw material costs ₹25 per unit. If output is 4,000 units, what are the total variable cost and the variable cost per unit?
- A) ₹25 total; ₹1,00,000 per unit
- B) ₹1,00,000 total; ₹25 per unit
- C) ₹1,00,000 total; per unit will fall
- D) ₹4,000 total; ₹25 per unit

> ✅ **Answer: B.** 25 × 4,000 = ₹1,00,000. Variable cost per unit stays constant. 📖 Section 3D

**Q3.** Telephone bill: ₹2,000 fixed rental + ₹3 per call. This month there were 1,500 calls. What is the total bill?
- A) ₹3,500
- B) ₹6,500
- C) ₹5,000
- D) ₹4,500

> ✅ **Answer: B.** 2,000 + (3 × 1,500) = 2,000 + 4,500 = **₹6,500**. This is a semi-variable cost. 📖 Section 3D

**Q4.** A factory's electricity bill is ₹70,000 at 4,000 machine-hours and ₹94,000 at 7,000 machine-hours. What are the variable rate per hour and the fixed component?
- A) ₹17.5/hr; ₹0
- B) ₹13.4/hr; ₹16,000
- C) ₹8/hr; ₹70,000
- D) ₹8/hr; ₹38,000

> ✅ **Answer: D.** Variable rate = (94,000 − 70,000) ÷ (7,000 − 4,000) = 24,000 ÷ 3,000 = ₹8. Fixed = 70,000 − (8 × 4,000) = ₹38,000. Check: 38,000 + 56,000 = 94,000 ✅. This is how a semi-variable cost is split into fixed and variable parts. 📖 Section 3D

**Q5.** How is a production supervisor's salary classified by traceability and by function?
- A) Direct cost; Administration
- B) Indirect cost (overhead); Production
- C) Direct cost; Production
- D) Indirect cost; Selling

> ✅ **Answer: B.** The supervisor oversees the whole factory and cannot be traced to any one unit, so it is an **overhead**. The work is in the factory, so it is a **production** cost. 📖 Sections 3B, 3C

**Q6.** What is lubricant oil for machines in a car factory called?
- A) Direct labour
- B) Selling cost
- C) Direct material
- D) Indirect material (overhead)

> ✅ **Answer: D.** Lubricant cannot be economically traced to any one car. This is the slide's own example. 📖 Section 3A

**Q7.** Classify sales commission (5% on each sale) by function and by behaviour.
- A) Distribution; Fixed
- B) Selling; Variable
- C) Production; Fixed
- D) Administration; Semi-variable

> ✅ **Answer: B.** It is spent to generate sales, so it is a **selling** cost. It rises with sales, so it is **variable**. 📖 Sections 3B, 3D

**Q8.** A company has already spent ₹50 lakh on R&D. Launching the product will need another ₹20 lakh, and the launch is expected to bring ₹30 lakh of contribution. What should it do?
- A) Not enough information to decide
- B) Don't launch, because total cost of ₹70L exceeds ₹30L
- C) Launch, because the incremental benefit is ₹10L; the ₹50L is a sunk cost
- D) Launch, because the ₹50L must be recovered

> ✅ **Answer: C.** The ₹50L will not come back under either option, so it is **sunk** and irrelevant. Look only at the future: 30 − 20 = **+₹10L**. Option B is the sunk cost fallacy. 📖 Section 3E

**Q9.** A company has an empty warehouse that it could rent out for ₹4 lakh/year, but it uses it to store its own stock. What is the opportunity cost of this decision?
- A) ₹4 lakh/year
- B) The warehouse's depreciation
- C) ₹0, because the warehouse is already owned
- D) The warehouse's purchase price

> ✅ **Answer: A.** The ₹4L from the next best alternative (renting it out) is given up. The warehouse's purchase price is **sunk**. 📖 Section 3E

**Q10.** A company must choose between two machines. Factory rent stays the same ₹2L under both options. Which cost is **relevant**?
- A) Factory rent
- B) The CEO's salary
- C) The different operating (power) cost of each machine
- D) Software bought last year

> ✅ **Answer: C.** A relevant cost is a **future** cost that **differs** between options. Rent and the CEO's salary stay the same (irrelevant), and the software is a sunk cost. 📖 Section 3E

**Q11.** The total cost of producing 5,000 units is ₹12,00,000, and of producing 5,200 units is ₹12,40,000. What is the marginal cost?
- A) ₹200
- B) ₹40,000
- C) ₹238.46
- D) ₹240

> ✅ **Answer: A.** ΔCost = 40,000 and ΔOutput = 200, so MC = 40,000 ÷ 200 = **₹200**. Option D (12,00,000 ÷ 5,000 = 240) is the average cost, and Option C (12,40,000 ÷ 5,200 ≈ 238.46) is also an average cost, not the marginal cost. 📖 Section 6

**Q12.** Budgeted overheads are ₹9,00,000 and budgeted machine-hours are 30,000. A product needs 4 machine-hours. How much overhead is absorbed by that product?
- A) ₹225
- B) ₹30
- C) ₹7.5
- D) ₹120

> ✅ **Answer: D.** OAR = 9,00,000 ÷ 30,000 = ₹30/machine-hour. Overhead absorbed = 30 × 4 = **₹120**. 📖 Section 5

**Q13.** A unit has DM of ₹350 and DL of ₹150. Overheads are absorbed at ₹25 per labour-hour, and one unit needs 3 labour-hours. What is the production cost per unit?
- A) ₹500
- B) ₹575
- C) ₹525
- D) ₹650

> ✅ **Answer: B.** Overhead = 25 × 3 = ₹75. Total = 350 + 150 + 75 = **₹575**. 📖 Section 5

**Q14.** SP is ₹500/unit, VC is ₹320/unit, fixed costs are ₹3,60,000 and 4,000 units are sold. What are the contribution per unit, total contribution and profit?
- A) ₹180; ₹7,20,000; ₹7,20,000
- B) ₹500; ₹20,00,000; ₹16,40,000
- C) ₹180; ₹7,20,000; ₹3,60,000
- D) ₹90; ₹3,60,000; ₹0

> ✅ **Answer: C.** 500 − 320 = ₹180. Total = 180 × 4,000 = ₹7,20,000. Profit = 7,20,000 − 3,60,000 = **₹3,60,000**. 📖 Section 6

**Q15.** Fixed cost is ₹5,00,000 and VC is ₹40/unit. At 10,000 units the total cost per unit is ₹90. If production rises to 12,500 units (within capacity), what is the total cost per unit?
- A) ₹80
- B) ₹90
- C) ₹112.5
- D) ₹72

> ✅ **Answer: A.** Fixed per unit = 5,00,000 ÷ 12,500 = ₹40, and VC = ₹40, so total = **₹80**. Per-unit cost falls because fixed cost is spread over more units. 📖 Section 3D

**Q16.** A factory has spare capacity. A product's normal price is ₹1,200, VC is ₹700 and full cost is ₹1,000 (including a ₹300 fixed share). A customer offers a one-time order at ₹850/unit. What should the factory do?
- A) Accept only if the price is ₹1,200
- B) Reject, because ₹850 < full cost of ₹1,000
- C) Accept, because each unit earns ₹150 contribution and fixed cost is incurred anyway
- D) Reject, because ₹850 < normal price

> ✅ **Answer: C.** With spare capacity, fixed cost is irrelevant and the minimum price = MC (₹700). 850 − 700 = **+₹150/unit**. This is like the airline's empty-seat case. 📖 Section 7, Case 1

**Q17.** A hotel has only 4 rooms free tonight. Walk-in guests pay ₹6,000/room-night, and variable cost is ₹1,500/room. A travel agent wants all 4 rooms at ₹3,500 each. What is the **minimum acceptable price**?
- A) ₹6,000
- B) ₹7,500
- C) ₹3,500
- D) ₹1,500

> ✅ **Answer: A.** There is a capacity constraint, so Min price = MC + Opportunity cost = 1,500 + (6,000 − 1,500) = **₹6,000**. Option B wrongly takes the opportunity cost as 6,000 (1,500 + 6,000); it should be the contribution (6,000 − 1,500 = 4,500). 📖 Section 7, Case 2

**Q18.** In Q17, if the hotel accepts the agent's offer, how much less contribution does it earn compared with walk-in guests?
- A) ₹4,000
- B) ₹14,000
- C) ₹18,000
- D) ₹10,000

> ✅ **Answer: D.** Contribution from the agent = 3,500 − 1,500 = ₹2,000/room, and from walk-ins = 6,000 − 1,500 = ₹4,500/room. Loss = 4 × (4,500 − 2,000) = **₹10,000**. 📖 Section 7, Case 2

**Q19.** How are fixed costs treated under the marginal costing technique?
- A) Converted into variable costs
- B) Treated as period costs and deducted directly from that period's contribution
- C) Absorbed into each unit and included in inventory
- D) Ignored completely

> ✅ **Answer: B.** Under marginal costing only variable cost goes to the product. Fixed cost is a period cost, so Profit = Contribution − Fixed. 📖 Section 6

**Q20.** Which industry–cost unit pair is **incorrect**?
- A) Electricity company: per kWh
- B) Hotel: per room-night
- C) Hospital: per patient-day
- D) Transport company: per passenger-kilometre

> ✅ **Answer: D.** A goods transport company's cost unit is **per tonne-kilometre**. Passenger-km is the cost unit for an airline (or bus). 📖 Section 4

**Q21.** A company buys a ₹10,00,000 machine that will last 5 years (straight-line depreciation). Which statement is correct for the first year?
- A) The ₹10L is a cost that becomes an asset; ₹2L depreciation is this year's expense
- B) Both cost and expense are ₹0
- C) The full ₹10L is this year's expense
- D) ₹10L is an expense, not a cost

> ✅ **Answer: A.** Every expense is a cost, but not every cost is an expense. Only this period's share (10L ÷ 5 = ₹2L) becomes an expense against revenue. 📖 Section 2

**Q22.** Which statement about **management accounting** is correct?
- A) It provides financial + non-financial info and is future-oriented, for planning, control and decisions
- B) It is strictly bound by GAAP and provides only financial info
- C) It prepares the annual report for outside investors
- D) It is only for tax calculation

> ✅ **Answer: A.** GAAP and external reporting belong to financial accounting. Management accounting is internal, flexible and future-focused. 📖 Section 1
