# Skill · Indian Income Tax Calculator

> **What this skill does:** Walks any Indian resident taxpayer through computing income tax for a given Assessment Year — from "I have nothing organised" to a defensible final liability + advance-tax plan. Output is a **single-page self-contained HTML tax-computation document** that mirrors a ClearTax-style ITR computation, with better UI, live 234B calculation, and ₹-prefixed tabular numerals throughout.
>
> **Works anywhere skills are supported** — web, desktop, and CLI-based tools alike.
>
> **Trigger phrases:** "compute my Indian tax", "calculate ITR", "AY 20XX-XX tax", "Old vs New Regime", "RSU / ESOP / ESPP tax India", "help me file my return".

---

## How the user supplies inputs (two-way)

The user can answer at any prompt by:

**(A) Replying in plain English** — describe their situation in a few sentences. The skill parses keywords and queues the right collection branches.

**(B) Dropping documents** — either way:
- **Filesystem-style tools:** drop files into a `docs/` folder in the working directory. The skill keeps re-reading that folder as more files appear over multiple turns.
- **Chat-style tools:** attach files directly in the conversation. Treat exactly like the filesystem case — same auto-detection logic.

Auto-detect from filename patterns (case-insensitive): `AIS*`, `26AS*`, `Form*16*`, `*interest*cert*`, `*P&L*` / `*PnL*`, `*RealisedGains*`, `*RSU*` / `*vest*`, `*ESPP*`, `*1099*`, `*Fidelity*` / `*Schwab*` / `*Etrade*`, `*Coin*` / `*WazirX*`, plus common Indian bank prefixes.

When files are dropped, **echo back what was detected** before continuing, e.g.:

```
✓ Detected from /docs:
   AIS_<...>.pdf            → foundation
   Form16_<...>.pdf         → foundation
   <Bank>_Interest_Cert.pdf → bank interest
   <Broker>_RealisedGains.pdf → capital gains

Anything else to add? Or shall I proceed with what's here?
```

**Locked / password-protected PDFs:** Many Indian fiscal docs (AIS, 26AS, Form 16, bank certs) are password-protected by default. If a file fails to read, ask the user to share an **unlocked version** OR (only if they're comfortable) the password. **Never persist passwords beyond the active session.**

**Unknown files in `docs/`:** if the filename doesn't match any known pattern, ask the user what it is before assuming.

---

## Operating principles

1. **Lowest set first.** Start with the foundation docs every taxpayer has. Escalate only based on the profiling answers.
2. **Plain-English over menus.** Use conversational profiling first; fall back to numbered gates only if the user's answer is ambiguous.
3. **Parse, don't interrogate.** Map natural-language answers silently to internal branches; only ask follow-ups for genuine ambiguity.
4. **Always reconcile, never trust a single source.** AIS lags Q4. Bank certificates beat AIS. Broker statements beat both for capital gains.
5. **Don't over-report.** If a line is genuinely zero, skip it. Don't invent income. Don't double-count.
6. **Confirm understanding before computing.** Echo a CLI summary; let the user veto any number before generating HTML.

---

## Conversation flow

### Step 0 · Quick triage (3 questions, one at a time)

```
Q1: Which AY are we computing for? (e.g., AY 2026-27 = FY 2025-26)
Q2: Are you salaried, freelancer/self-employed, both, or business owner?
Q3: Approximate gross income range — < ₹5 L / ₹5–15 L / ₹15–50 L / > ₹50 L?
```

This determines:
- **ITR form** (1 / 2 / 3 / 4)
- **Schedule AL** mandatory? (income > ₹50 L)
- **Advance tax** obligation? (Sec 208 — if liability ≥ ₹10 K)
- **Tax audit** risk? (re-checked in Step 0.5 if business)

---

### Step 0.5 · Plain-English Profiling (the smart core)

Ask **all five questions in one prompt**, asking the user to reply in a few short sentences:

```
Tell me a bit about your year — answer these in any order, in plain language:

  1. Work · Are you salaried, a freelancer, or running a business?
            Did you change jobs this year or receive salary arrears?

  2. Real estate · Are you paying an EMI on a home loan (even an under-construction one),
            paying rent, or receiving rent?

  3. Company stock & foreign · Do you receive RSUs / ESPPs from your employer?
            Do you hold any foreign stocks, brokerage accounts, or bank accounts?

  4. Trading & investments · Did you SELL any shares, mutual funds, or crypto this year?
            Or do you just have bank savings / FDs?

  5. Side income & extras · Any side-gigs, freelance work, large gifts (> ₹50 K from
            non-relatives), or matured policies (PPF / EPF / insurance) paid out this year?
```

#### Internal mapping logic — how the skill parses answers

Run these silently against the user's reply. Trigger the corresponding branch in Step 2.

| User said something like | Trigger branch / flag |
|---|---|
| "salaried at <X>" / "I work for a company" | Step 2.1 (Salary) — request Form 16 / ITCS |
| "changed jobs", "switched companies", "two employers", "received arrears" | + Sec 89 / Form 10E flag · request **all** Form 16s |
| "freelance", "consulting", "self-employed", "own business", "GST registered" | Step 2.7 (Business / Profession). Also ask gross receipts → Tax Audit check |
| "GST"-aware or "turnover > ₹X" | Probe for Tax Audit u/s 44AB thresholds |
| "EMI", "home loan", "principal", "interest paid on loan" | Step 2.2 (House Property) — Old Regime relevance flag |
| "rent paid", "I pay rent", "₹X / month rent" | HRA exemption check (Old Regime); request rent receipts + landlord PAN if rent > ₹1 L/yr |
| "rent received", "tenant", "let out", "rental income" | Step 2.2 (House Property — let-out) |
| "RSU", "vested", "stock units", "Restricted Stock" | Step 2.4 (RSU) — request vest letters + custodial broker statement |
| "ESPP", "purchase plan", "stock purchase discount" | Step 2.5 (ESPP) — different valuation logic from RSU |
| "Fidelity", "E*Trade", "Schwab", "foreign brokerage" | Step 2.4 / 2.5 + **Schedule FA mandatory flag** |
| "foreign dividend", "US dividend", "ORCL dividend", "WHT" | Step 2.6 (Other Sources — foreign income) + FTC / Form 67 flag |
| "sold shares", "redeemed MF", "capital gains", "STCG", "LTCG" | Step 2.3 (Capital Gains — Indian equity / MF) |
| "crypto", "bitcoin", "VDA", "WazirX", "CoinDCX" | Step 2.8 (Crypto / VDA — Sec 115BBH 30%) |
| "intraday", "BTST", "speculative" | Step 2.9 (Speculative business) — flag potential ITR-3 |
| "bank interest", "FD", "fixed deposit", "savings interest", "RD" | Step 2.6 (Other Sources — bank deposits) |
| "Indian dividend", "MF dividend", "company dividend", "194K TDS" | Step 2.6 (Other Sources — Indian dividend) |
| "gift", "received money from", "wedding gift > ₹50 K" | Step 2.10 (Gifts u/s 56(2)(x)) |
| "agricultural", "farm income" | Step 2.10 (Agricultural income — partial integration if > ₹5 K) |
| "PPF maturity", "insurance payout", "EPF withdrawal", "Sukanya" | Step 2.10 (Exempt income — still requires AIS-side disclosure) |
| "previous year loss", "carry forward" | Step 2.11 (Schedule CFL) — request prior ITR |
| "more than ₹50 L income" / "high net worth" / "Schedule AL" | Step 2.12 (Schedule AL — full asset disclosure) |

If the user's answer is genuinely unclear, fall back to numbered triage gates (see Appendix A).

---

### Step 1 · Foundation documents (always)

Ask unconditionally:

```
The foundation set — please share these (drop into /docs or attach in chat):

  1. AIS PDF        — incometax.gov.in → Services → Annual Information Statement
  2. Form 26AS PDF  — TRACES portal (or via incometax.gov.in)
  3. Form 16 / employer salary computation (if salaried)
     OR Books / P&L (if self-employed / business)
```

Where to download (mention only if user asks):
- **AIS:** `incometax.gov.in` → Services → Annual Information Statement → choose AY → Download
- **26AS:** `incometax.gov.in` → My Account → View Form 26AS → redirected to TRACES → choose AY
- **Form 16 / Tax Computation Statement:** Employer payroll portal — search "Form 16" or "Tax Computation"

If the user **changed jobs** during the FY → ask for **all** Form 16s (one per employer).

If **password-locked** → ask for unlocked version OR password (don't persist).

Don't proceed without the foundation set.

---

### Step 2 · Branched data collection

Activated branches per the profiling logic. Order: bank → capital gains → stocks → RSUs → ESPP → other sources → business → speculative → crypto → gifts/exempt → CFL → Schedule AL.

#### 2.1 Salary (always for salaried)

```
Confirmed from Form 16 / ITCS:
  • 17(1) breakdown — Basic, HRA, allowances, dividend equivalents
  • 17(2) perquisites — RSU vest, rent-free accommodation, etc.
  • Standard Deduction (auto): ₹75 K (New) / ₹50 K (Old)
  • TDS by employer (per quarter)

Multiple Form 16s? Combine all.

If salary arrears were paid, please share:
  • Year-wise breakdown of the arrears
  • Each prior year's taxable salary
  → Section 89 relief computation needed; Form 10E to be filed BEFORE the return.
```

#### 2.2 House Property

```
For each property:
  • Self-occupied or let-out?
  • If let-out: monthly rent, municipal tax paid, vacancy period
  • Home loan? Share Interest Certificate u/s 24(b) and Principal cert u/s 80C (Old Regime)
  • If you pay rent (HRA-eligible, salaried, Old Regime):
    – Total rent paid
    – Landlord PAN if rent > ₹1 L/year (mandatory)
```

#### 2.3 Capital gains — Indian shares / MF

```
From your broker (any):
  • Annual P&L / Capital Gains report (full year)
  • Tradewise statement / AGTS — needed if redoing reconciliation
  • Mutual fund consolidated statement (CAMS / KFinTech)

Cross-checks:
  • Equity-oriented MF (STT-paid): 111A (20% STCG) or 112A (12.5% LTCG over ₹1.25 L)
  • Debt MF purchased ≥ 1-Apr-2023: slab rate (Sec 50AA)
  • Debt MF purchased pre-1-Apr-2023, held > 24 mo: 12.5% LTCG no indexation
  • Gold / Silver ETF: slab if ≤ 24 mo, 12.5% LTCG if > 24 mo
```

#### 2.4 RSUs / Foreign Stock Plan

```
Please share:
  • Custodial broker statement — Custom Transaction Summary / 1099 / Stock Plan history
  • RSU Vest Letter from employer (per vest event) — has Merchant Banker FMV
  • Inward Remittance certificates from your bank (FCY → INR transfers)

Tax treatment:
  • At vest → perquisite under 17(2), taxed as salary at FMV (employer deducts TDS)
  • On sale → capital gain over the same FMV (cost basis); no double tax
  • Foreign-listed (no Indian STT): STCG at slab if ≤ 24 mo; LTCG @ 12.5% if > 24 mo

Schedule FA (calendar-year basis, e.g., CY 2025 for AY 2026-27):
  • MANDATORY — even if you held one share for one day during CY
  • Penalty: ₹10 L per omitted asset (Black Money Act)
  • Need: opening + peak + closing balances, dividends, sale proceeds
```

#### 2.5 ESPP (Employee Stock Purchase Plan)

```
ESPP differs from RSU:
  • At purchase: perquisite = DISCOUNT (FMV on purchase date − discounted price paid),
    taxed as 17(2) salary (employer deducts TDS)
  • At sale: cost basis = Discounted price you paid + Perquisite value already taxed
                       = FMV on purchase date (algebraic equivalence)
    Using FMV-on-purchase-date as cost basis is the cleanest convention; use the
    employer's reported basis on the broker 1099 if it matches.

Please share:
  • ESPP purchase confirmation / contribution period summary
  • Employer's perquisite TDS statement (if separate from regular salary)
  • Custodial broker statement (post-purchase holdings)
  • If sold: same STCG / LTCG rules as RSU (foreign listed → slab/12.5% by holding period)
```

#### 2.6 Income from Other Sources

```
Please share whatever applies:
  • Bank Interest Certificate (per bank, full year — savings / FD / RD)
  • Year-end statement (31-Mar) for closing balances (if Schedule AL applies)
  • Form 16A (TDS certificate) if interest crossed ₹50 K (₹1 L for senior citizens)
  • Indian dividend statements (broker app exports them; > ₹5 K MF dividend → 10% TDS u/s 194K)
  • Foreign dividend / interest (custodial / Schedule FSI for FTC)
  • Joint accounts: who is the contributor? Default 100% to certificate-holder.
```

#### 2.7 Business / Profession

```
Share:
  • P&L + Balance Sheet (full year)
  • All Form 16As from clients
  • Bank statements showing all professional receipts
  • Expense receipts (rent, utilities, depreciation, etc.)
  • Decision: Presumptive 44ADA (50% of receipts) vs full books?

CRITICAL — Tax Audit threshold check (u/s 44AB):
  • Gross receipts > ₹75 L for professionals (44ADA threshold)?
  • Gross turnover > ₹3 Cr for businesses (44AD digital receipts) / ₹2 Cr otherwise?
  → Mandatory Tax Audit; CA sign-off required; due date moves to 31-Oct.
  → Auto-escalate to CA recommendation.
```

#### 2.8 Crypto / VDA

```
Please share:
  • Annual statement from VDA exchange (CoinDCX / WazirX / Binance / etc.)
  • All buy/sell trades for Schedule VDA (each transaction reported separately)

Tax: flat 30% u/s 115BBH; 1% TDS u/s 194S on each disposal.
Losses: cannot be set off against any other income; cannot be carried forward.
```

#### 2.9 Speculative business (intraday)

```
Same-day buy + sell of equity (no delivery) is speculative business u/s 43(5).
Even small amounts force ITR-3 (instead of ITR-2).

If speculative LOSS < ₹500 (de minimis) and no speculative income:
  • Pragmatic call: drop it, stay in ITR-2.
  • Trade-off: forgo carry-forward of the tiny loss.
  • CAs accept this for trivial amounts; document the decision.

If speculative profit, OR loss is material → must declare → ITR-3.
```

#### 2.10 Gifts & exempt incomes (Sec 56(2)(x), agricultural, maturities)

```
Please tell me about:
  • Gifts from non-relatives > ₹50 K aggregated in the FY?
    → Taxable as Other Sources at slab rate
    → Gifts from relatives (defined list) and on occasion of marriage are exempt
  • Agricultural income > ₹5 K?
    → Declare for "rate purposes" (partial integration); not directly taxable
  • PPF / Sukanya / EPF / NPS Tier-II / LIC maturity in this FY?
    → Most are exempt u/s 10(11) / 10(11A) / 10(10D)
    → BUT they appear in AIS — must be disclosed in Schedule EI to prevent mismatch notice

Even exempt = report in Schedule EI. Notice prevention > brevity.
```

#### 2.11 Brought-forward losses

```
Share last year's (or earlier) ITR-V / Form_pdf showing Schedule CFL.
Limitations:
  • STCL: 8 yrs, against any STCG / LTCG
  • LTCL: 8 yrs, against LTCG only
  • Speculative loss: 4 yrs, against speculative income only
  • Business loss (non-speculative): 8 yrs, against business income
  • House property loss: 8 yrs (current year set-off capped at ₹2 L vs salary)
```

#### 2.12 Schedule AL — full asset declaration (when income > ₹50 L)

If applicable, ask for **all** of these (don't stop at bank balances):

```
Schedule AL is mandatory because total income > ₹50 L.
For values as on 31-Mar of the FY, please share / state:

  Movable assets:
    • Bank deposits — closing balances per account (savings / FD / RD)
    • Mutual fund holdings — at cost or current value
    • Listed shares — at cost or current value
    • Unlisted shares / private equity — book value
    • Insurance policies — sum assured or surrender value
    • Vehicles, yachts, boats, aircraft — purchase price
    • Jewellery, bullion — at cost (or fair value if recently appraised)
    • Cash in hand — running balance

  Immovable assets:
    • Land, residential property, commercial property — purchase price + stamp duty + registration

  Liabilities:
    • Home loan outstanding
    • Other loans (personal, auto, business) outstanding

  Loans & Advances given:
    • Money lent to friends / family / business — outstanding receivable
```

---

### Step 3 · Old vs New Regime (only if material)

Skip this entirely if user's income < ₹12 L (New Regime always wins via 87A) OR if no significant 80C / 80D / 80CCD / HRA / home-loan-interest exist.

Otherwise:

```
For the comparison I'll need:
  • Rent paid (HRA exemption — only Old)
  • 80C investments (PPF, ELSS, LIC, EPF, principal repayment, kid tuition) — max ₹1.5 L
  • 80D health insurance premium — max ₹25 K + ₹50 K (parents)
  • 80CCD(1B) NPS Tier-I additional — max ₹50 K
  • 80E education loan interest — no limit
  • 80G donations
  • Home loan interest u/s 24(b) — max ₹2 L self-occupied
```

Run both regimes; present a comparison table; recommend the lower.

---

### Step 4 · Reconcile (de-duplicate)

Show a reconciliation table — every "no double-count" decision:

| Item | AIS shows | Certificate / Broker shows | Final amount used | Why |
|---|---|---|---|---|
| Salary | ₹X (Q1-Q3 only) | ₹Y (Form 16 full year) | ₹Y | AIS lags; Form 16 = full year |
| Bank interest | ₹A (TDS-trigger only) | ₹B (full year cert) | ₹B | Cert > AIS |
| TDS – salary | ₹P | ₹Q | ₹Q | Form 16 authoritative |
| ... | | | | |

Echo back; let user veto any line.

Items to **deliberately exclude** (no over-reporting):
- Inward remittances (not income — already counted as broker sale proceeds)
- Foreign dividends in AIS for prior FY (out of scope this year)
- Speculative loss < ₹500 (de minimis, per Step 2.9 if user opted out)

---

### Step 5 · Compute (CLI summary first, then HTML)

Before generating the HTML, present a CLI summary in plain text:

```
══════════════════════════════════════════════════════════════
  HEAD-WISE INCOME (per reconciled values)
══════════════════════════════════════════════════════════════
[1] SALARY                                  ₹XX,XX,XXX
    (Section 17(1)  · ₹..)
    (Section 17(2)  · ₹..)
[2] HOUSE PROPERTY                          ₹XX,XXX
[3] CAPITAL GAINS                           ₹XX,XXX
    (111A · ₹.. @ 20%)
    (Slab · ₹..)
    (LTCG · ₹..)
[4] OTHER SOURCES                           ₹XX,XXX
    (Indian interest / dividend / foreign)
[5] BUSINESS / PROFESSION (if any)          ₹XX,XXX

GROSS TOTAL INCOME                          ₹XX,XX,XXX
Less: Chapter VI-A                          (₹..)
TOTAL INCOME (Sec 288A rounded)             ₹XX,XX,XXX

  Slab tax       ₹..
  + 111A tax     ₹..
  + LTCG tax     ₹..
  + Surcharge    ₹..
  + Cess @ 4%    ₹..
TOTAL TAX                                   ₹XX,XX,XXX

Less: TDS Salary                            (₹..)
Less: TDS Other                             (₹..)
Less: Advance Tax / SAT                     (₹..)
Less: FTC u/s 90                            (₹..)
NET TAX BEFORE INTEREST                     ₹XX,XXX
+ 234B (1% × n months)                      ₹..
+ 234C (post Sec 234C(1) proviso, if any)   ₹..

💰 TOTAL PAYABLE / (REFUND)                  ₹XX,XXX
══════════════════════════════════════════════════════════════
```

Get user confirmation. Address any flagged questions. Then generate HTML.

---

### Step 6 · Output — single-page HTML

The skill's deliverable is a **single self-contained HTML page** (`tax-computation.html`) that mirrors a ClearTax-style ITR Computation PDF but better:

Required sections, top to bottom:

1. **Header** — name, PAN, AY, FY, regime, ITR form
2. **Tax Summary** card (4-up: Total Income · Total Tax · TDS+FTC · Net Payable)
3. **Salary Income** (collapsible) — 17(1) / 17(2) breakdown, Std Deduction
4. **House Property** (only if applicable) — let-out / self-occupied, interest u/s 24(b)
5. **Income from Other Sources** (collapsible) — savings / FD-RD / dividend / foreign, account-by-account
6. **Capital Gains** (collapsible) — by bucket (111A / slab / 112A / 112), per-instrument detail
7. **Tax Computation** (collapsible) — slab waterfall visualisation, 111A, surcharge, cess
8. **TDS & Credit Reconciliation** (collapsible) — AIS vs certificates, FTC
9. **Final Payable** (collapsible) — with target-date picker for live 234B
10. **Schedule FA** (only if foreign asset/income)
11. **Schedule AL** (only if income > ₹50 L)
12. **Schedule EI** (only if exempt income reported)
13. **Reconciliations & items not reported** (collapsible)

Visual baseline:
- Clean white background, refined editorial type
- Sans body (Plus Jakarta Sans or Manrope) + serif headings (Fraunces or similar)
- JetBrains Mono for tabular numerals
- Restrained accents (oxblood + gold)
- ₹ prefix on all monetary values consistently

**Do NOT generate** historical / insights / future-projection tabs — those are optional personal extras, not part of this skill's output.

---

## Tax rate quick-reference (FY 25-26 / AY 26-27)

### Slabs · New Regime (default)

| Slab | Rate |
|---|---|
| ₹0 – 4 L | 0% |
| ₹4 – 8 L | 5% |
| ₹8 – 12 L | 10% |
| ₹12 – 16 L | 15% |
| ₹16 – 20 L | 20% |
| ₹20 – 24 L | 25% |
| Above ₹24 L | 30% |

Std Deduction: **₹75,000** · 87A Rebate: **up to ₹60,000 if TI ≤ ₹12 L** · Surcharge cap: **25% in New Regime**.

### Slabs · Old Regime (optional)

| Slab (non-senior) | Rate |
|---|---|
| ₹0 – 2.5 L | 0% |
| ₹2.5 – 5 L | 5% |
| ₹5 – 10 L | 20% |
| Above ₹10 L | 30% |

Std Deduction: **₹50,000** · 87A: **up to ₹12,500 if TI ≤ ₹5 L** · Chapter VI-A allowed.

### Surcharge bands (income excl. 111A/112A)

| Income | Old Regime | New Regime |
|---|---|---|
| ≤ ₹50 L | 0% | 0% |
| ≤ ₹1 Cr | 10% | 10% |
| ≤ ₹2 Cr | 15% | 15% |
| ≤ ₹5 Cr | 25% | 25% |
| > ₹5 Cr | 37% | 25% (capped) |

Cess: 4% Health & Education in both regimes.

### Capital gains rates (post 23-Jul-2024)

| Asset | STCG / LTCG threshold | STCG rate | LTCG rate |
|---|---|---|---|
| Listed Indian equity / equity-MF (STT paid) | 12 mo | 20% (Sec 111A) | 12.5% over ₹1.25 L (Sec 112A) |
| Foreign equity (no Indian STT) | 24 mo | Slab | 12.5% no indexation (Sec 112) |
| Debt MF (purchased ≥ 1-Apr-2023) | always slab | Slab | Slab (Sec 50AA) |
| Debt MF (purchased pre-1-Apr-2023) | 24 mo | Slab | 12.5% no indexation |
| Gold / Silver ETF | 24 mo | Slab | 12.5% no indexation |
| Real estate | 24 mo | Slab | 12.5% no indexation |
| Crypto / VDA | always 30% | 30% (Sec 115BBH) | 30% |

### 80-section quick-reference (Old Regime · selected)

| Section | Limit | What |
|---|---|---|
| 80C | ₹1,50,000 | EPF, PPF, ELSS, LIC, home loan principal, tuition |
| 80CCD(1B) | ₹50,000 | NPS Tier-I additional (over 80C) |
| 80CCD(2) | up to 10% Basic+DA | Employer NPS — **also allowed in New Regime** |
| 80D | ₹25,000 (₹50,000 senior) | Health insurance premium |
| 80E | no limit | Education loan interest |
| 80G | varies | Charitable donations |
| 80TTA / 80TTB | ₹10 K / ₹50 K | Savings / deposit interest (Old only) |
| 24(b) | ₹2 L self-occupied | Home loan interest |

### Advance tax instalment schedule (Sec 211 · individuals)

| Due date | Cumulative % to be paid |
|---|---|
| 15-Jun (Q1) | 15% of estimated tax-after-TDS |
| 15-Sep (Q2) | 45% cumulative |
| 15-Dec (Q3) | 75% cumulative |
| 15-Mar (Q4) | 100% cumulative |

Sec 234C interest = 1% per month (3 months Q1-Q3 · 1 month Q4) on the shortfall, rounded down to the nearest ₹100.

**Sec 234C(1) proviso:** Capital gains / dividend / Sec 115BBDA income arising **after** an instalment due date is exempt from interest for that and all earlier instalments — provided the tax on it is paid in the next instalment due after the income arose.

### Tax-audit thresholds (u/s 44AB)

| Filer | Threshold | Trigger |
|---|---|---|
| Profession (44ADA) | Gross receipts > ₹75 L | Tax audit mandatory |
| Business (44AD, ≥ 95% digital) | Turnover > ₹3 Cr | Tax audit mandatory |
| Business (regular) | Turnover > ₹2 Cr | Tax audit mandatory |
| Either, if claiming lower-than-presumptive profit | Per Sec 44AB(d)/(e) | Audit mandatory |

Tax audit → ITR due date moves to 31-Oct → CA sign-off required → ESCALATE to professional review.

---

## Common pitfalls (always remind the user)

1. **AIS shows Q1-Q3 in early May** — Q4 TDS appears after 31-May. Use Form 16 / certificates as authoritative.
2. **Schedule FA penalty** — ₹10 L per omitted foreign asset under Black Money Act. Disclose even tiny holdings.
3. **Form 67** is a pre-condition for FTC u/s 90 — file BEFORE the return, not after.
4. **Form 10E** for Sec 89 arrears relief — must be filed BEFORE the return.
5. **RSU perquisite** is taxed as salary at vest. Sale gain is computed over the same FMV. Never double-count.
6. **ESPP** perquisite is the DISCOUNT at purchase. Cost basis = FMV at purchase, not the discounted price paid.
7. **Intraday equity trades** are speculative business income — even ₹1 of profit forces ITR-3.
8. **Joint bank accounts** — tax follows certificate-holder PAN unless explicit split with contribution evidence.
9. **Old vs New regime** — re-check every Budget; thresholds and rates change.
10. **Inward remittances** are not income — they're transfers; the underlying income (sale / dividend) is already counted.
11. **Cost basis for RSU** = Merchant Banker FMV in INR (per vest letter), NOT broker's USD market price × FX (legally).
12. **Section 234C(1) proviso** — late-arising CG / dividend gets relief from earlier instalments' interest. Apply when CG sale was post the relevant Q due date.
13. **Schedule EI** — exempt income (PPF maturity, life insurance proceeds, agricultural) must still be disclosed to prevent AIS-mismatch notices.
14. **Tax audit thresholds** — gross receipts > ₹75 L (profession) or turnover > ₹3 Cr / ₹2 Cr (business) → mandatory CA audit.

---

## When to escalate (auto-recommend a CA / professional)

- **NRI / RNOR / US person under FATCA** → flag and recommend a CA.
- **Tax audit triggered** (Sec 44AB) → mandatory CA sign-off, escalate.
- **Tax liability > ₹10 L** with non-trivial CG / business income → recommend CA review before filing.
- **Black Money Act exposure** (undisclosed foreign asset > 2 years) → strongly recommend professional intervention.
- **Search / seizure or notice received** → mandatory CA + lawyer.
- **Estate / inheritance / gift > ₹50 L** with valuation complexity → CA recommended.
- **Multiple PANs / amended returns / prior-year defects** → CA recommended.

---

## Output format

Single file: **`tax-computation.html`** — self-contained, no server, no build step. Open in any browser.

The user can re-run the skill next year by dropping fresh documents into the same `docs/` folder (or attaching them in a new chat). The skill is deterministic given the same inputs.

---

## Appendix A · Numbered fallback gates

If the user's plain-English answer in Step 0.5 is genuinely ambiguous, fall back to these three gates (one at a time, multi-select):

**Gate 1 — Core Investments & Assets**
```
Reply with numbers (e.g., "1, 3") or "None".
  1. Bank or Post Office interest (Savings, FDs, RDs)
  2. Capital gains — Indian shares / Mutual Funds
  3. Real estate — rental income or home loan interest
  4. Indian dividend income
```

**Gate 2 — Complex & Foreign Assets**
```
  1. Foreign assets — RSUs, ESPPs, foreign shares / dividends
  2. Crypto / Virtual Digital Assets
  3. Brought-forward losses from previous AYs
```

**Gate 3 — Business, Other Sources & Edge Cases**
```
  1. Freelance / consulting / business income
  2. Intraday equity trading (speculative business)
  3. Gifts > ₹50 K, agricultural income, exempt payouts (PPF / insurance maturity)
  4. Salary arrears (requires Form 10E / Sec 89 relief)
```

---

*End of skill. Update annually after the Finance Act publishes new rates / thresholds.*
