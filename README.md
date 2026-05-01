# Indian Income Tax Calculator — AI Skill

A standalone, shareable AI skill that walks any Indian resident taxpayer through computing income tax for a given Assessment Year — from "I have nothing organised" to a clean, defensible single-page HTML tax-computation document.

### 📥 [Download the Latest Release (.zip)](https://github.com/aakarshsingh/indian-tax-calculator/releases/download/v1.0.0/indian-tax-calculator.zip)

---

## 🔒 Privacy & Data Security First
**Before you upload your AIS, Form 26AS, or Form 16 to any cloud-based AI:**
Please take a moment to redact or black out your Personally Identifiable Information (PII). The AI does not need your **PAN, Aadhaar number, or exact bank account numbers** to compute your taxes.

---

## Quick Start
1. **Download and Extract:** Download the [latest .zip release](https://github.com/aakarshsingh/indian-tax-calculator/releases/download/v1.0.0/indian-tax-calculator.zip) and extract it to your computer.
2. **Open your AI:** Start a new chat in your preferred AI tool (ChatGPT, Claude, Gemini, etc.).
3. **Upload the Skill:** Drag and drop the extracted `SKILL.md` file into the chat.
4. **Trigger the Calculator:** Type: *"Run the tax calculator skill."*
5. **Follow Along:** Answer the plain-English questions about your year and drop your PDF tax documents into the chat when asked.

*For CLI / Agent SDK users:* Extract the zip and place `SKILL.md` into your agent's skills or system prompt directory. Drop your documents into your agent's designated `docs/` folder.

---

## What this skill produces

**One file:** `tax-computation.html` — self-contained, no server, no build step. It mirrors a ClearTax-style ITR computation but with a refined editorial UI:
- 4-up tax summary card (Total Income · Total Tax · TDS+FTC · Net Payable)
- Collapsible per-head breakdowns (Salary · House Property · Capital Gains · Other Sources · Business)
- Slab-tax waterfall visualisation
- TDS reconciliation table (AIS vs certificates)
- Live 234B interest calculation (target-date picker)
- Schedule FA / AL / EI sections (only when applicable)
- All monetary values prefixed with ₹, tabular numerals, refined typography

---
## 👁️ Preview the Output

Curious what the final result looks like? Check out the live interactive demo hosted on GitHub Pages:

👉 **[View the Live Mock Computation](https://aakarshsingh.github.io/indian-tax-calculator/)**

---
## What the skill covers

### Income heads
- Salary (multiple Form 16s if you changed jobs)
- House property (self-occupied, let-out, home loan interest)
- Capital gains (Indian listed equity, equity-MF, debt MF, gold/silver, real estate, foreign-listed equity, crypto)
- Income from Other Sources (savings + FD/RD interest, Indian dividends, foreign dividends, gifts > ₹50 K, agricultural)
- Business / Profession (with tax-audit u/s 44AB threshold check)
- RSU perquisite (Sec 17(2), Merchant Banker FMV) and ESPP discount

### Special situations handled
- Section 89 / Form 10E relief for salary arrears
- Section 234B + 234C(1) proviso for late-arising capital gains
- Foreign Tax Credit u/s 90 (Form 67 pre-filing reminder)
- Old Regime vs New Regime side-by-side comparison
- Schedule FA (foreign assets, calendar-year basis)
- Schedule AL (mandatory if income > ₹50 L)
- Schedule EI (exempt income — to prevent AIS-mismatch notices)
- Schedule CFL (carry-forward losses from prior AYs)
- Joint bank accounts (default 100% to certificate-holder)
- De-minimis intraday loss handling (drop or declare based on materiality)

### Auto-escalation to a CA
The skill flags and recommends professional review when:
- NRI / RNOR / US person under FATCA
- Tax audit triggered (Sec 44AB)
- Tax liability > ₹10 L with non-trivial CG / business
- Black Money Act exposure (undisclosed foreign asset > 2 years)
- Notices / search / seizure received
- Multiple PANs, amended returns, prior-year defects

---

## What the skill does NOT do
- Does not file the return for you (you still e-verify on `incometax.gov.in`)
- Does not generate JSON / CSV exports (HTML only — by design)
- Does not give legal advice — the output is a tax-computation document, reviewed and filed at your discretion

---

## Contributing & Updating

Tax rules change every year with the Union Budget. If you notice an outdated rate, please feel free to fork the repository, update the rates, and open a Pull Request! 

When updating for a new Finance Act, check these sections in `SKILL.md`:
- Slab tables (New + Old)
- Surcharge bands
- Capital gains rates (especially post-budget changes)
- 80-section limits
- Tax-audit thresholds
- Sec 87A rebate cap

---

## Caveats

This skill is a **calculator and document generator**, not a substitute for a Chartered Accountant. For complex situations (NRI status, business audits, large foreign assets, prior-year notices, M&A consideration shares), engage a CA. The skill auto-flags these.

The output is your computation memo — you're responsible for the values you confirm and the return you ultimately file.

---

## Repository Structure
```text
/
├── indian-tax-calculator.zip            (Compiled release for end users)
├── SKILL.md                             (The core skill logic and tax rules)
└── README.md                            (This file)
```
---

## Tool-Specific Installation Guide

This skill is just a Markdown file (`SKILL.md`), meaning it works anywhere you can provide system instructions or upload files. Here is how to install it in your preferred tool:

### 🟧 Claude (Web & Desktop)
**Best method:** Projects
1. Create a new Project in Claude.
2. Upload `SKILL.md` to the **Project Knowledge**.
3. *Optional:* Set the Custom Instructions to: `Always use the Indian Income Tax Calculator skill when asked about taxes.`
4. Start a chat inside the project and say: *"Compute my tax."*

### 🟩 ChatGPT (Plus / Team / Enterprise)
**Best method:** Custom GPTs
1. Go to **Explore GPTs** -> **Create**.
2. Open the **Configure** tab.
3. Name it "Indian Tax Calculator".
4. Open `SKILL.md` in a text editor, copy all the text, and paste it into the **Instructions** box.
5. Save and start chatting.

### 🟦 Gemini Advanced / Google AI Studio
1. **Gemini Advanced (Web):** Start a new chat, upload the `SKILL.md` file, and type: *"Read this skill and compute my tax."*
2. **Google AI Studio:** Paste the contents of `SKILL.md` directly into the **System Instructions** box, then chat normally in the prompt tester.

### 🟨 CLI Agents (Claude Code, etc.)
Drop `SKILL.md` into your agent's system prompt directory or pass it directly on launch:
```bash
claude --system-prompt "$(cat SKILL.md)"
