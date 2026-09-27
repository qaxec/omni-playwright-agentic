---
name: test-case-writing
description: Input-readiness gate plus rules and format for writing or reviewing manual/golden test cases (ParaBank or any web app) — checks that a story/doc/screenshot is adequate before cases are generated, then preconditions with data-creation method, test data, UI-only steps, per-step verifiable expected results. Use whenever validating requirements for testability, or writing, reviewing or converting test cases.
---

# Writing test cases

Two phases, which can be run by two different agents:
1. **Step 0, input readiness:** check that the inputs are good enough and produce a readiness report. Stop here if the verdict is NOT READY.
2. **Writing:** turn the report's fact sheet into cases, using the format and rules below.

The reference example is [docs/test-cases/parabank-baseline.md](../../../docs/test-cases/parabank-baseline.md). Match its layout.

---

## Step 0: Input readiness (a gate before any case is written)

**Goal:** prove that every expected result the cases will need can be traced to a trustworthy source. If it can't, stop and ask. **Never invent requirements to fill a gap.**

### 0.1 List the inputs
List every source you were given or found, with this table:

| Source ID | Type | What it is | Date / version | Normative? |
|---|---|---|---|---|
| SRC-1 | story | PB-3 "Open new account", 4 ACs | 2026-09-26 | yes |
| SRC-2 | screenshot | Open New Account page | 2026-09-25 | no (shows current behaviour) |

- **Normative** means the source says what *should* happen: a story, AC, spec or a decision from the user.
- **Descriptive** means it shows what *does* happen: a screenshot, the running app, an API response.
- Only a normative source can prove that behaviour is correct.

### 0.2 What each source type usually gives, and what it lacks

| Source | Usually gives | Usually lacks | Fill the gap by |
|---|---|---|---|
| User story + AC | Actors, rules, scope | Exact UI text, error messages, data setup, formats | Explore the app, ask |
| Spec / design doc | Rules, fields, sometimes messages | Whether the running build matches it; data setup | Explore the app |
| Screenshot | Labels, layout, one happy-path state | Rules, limits, errors, what happens next, hidden fields | Ask for the story; explore |
| Running app / API docs | Everything observable: text, URLs, formats, setup endpoints | **Whether it is correct** | A normative source or the user |
| Verbal explanation from the user | Intent, flows, priorities | Exact values and wording | Explore, then confirm |

### 0.3 Readiness checklist
For each item, record PASS, FAIL or N/A, and the source tag (0.4). Items marked **[B]** are **blocking**: if one fails, the verdict is NOT READY.

**A. Scope and actors**
- **A1 [B]** The feature under test is named. What is in scope and what is out of scope are both written down.
- **A2** Actors and roles are known, including permission differences (for example admin vs customer).
- **A3 [B]** Entry points are known: the UI page or menu path, and any API or endpoint involved.
- **A4** The dependencies on other features are known (for example "Open Account needs an existing funded account").

**B. Behaviour and business rules**
- **B1 [B]** Each acceptance criterion or rule is **atomic** (one behaviour) and **checkable as true or false**.
- **B2 [B]** Every rule has **explicit values**: limits, amounts, counts, lengths. No vague words: "valid", "appropriate", "reasonable", "quickly", "etc.", "and so on", "should handle".
- **B3** Every calculation has a formula, a precision or rounding rule, and a currency or unit.
- **B4** State changes are listed: which entities change, and **every place the change should appear** (pages, dropdowns, totals, history, other accounts).
- **B5** Conflicts between sources are listed and resolved. If they can't be resolved, they are questions.
- **B6** The order or sequence rules are clear: what must happen before what, and whether a step can be repeated.

**C. Inputs and boundaries**
- **C1** Every input field is described: type, required or optional, format, allowed values or range, default value.
- **C2** Boundaries are defined, including whether they are **inclusive or exclusive** (is `= 100` allowed?).
- **C3** Invalid input handling is defined for each field or rule: rejected, corrected or ignored, and with which message.

**D. Observable outcomes (UI and API facts)**
- **D1 [B]** The exact success indicator for each action is known: heading, text, URL or status code.
- **D2 [B if negative paths are in scope]** The exact error message is known for each error condition, and where it appears.
- **D3** The destination after each action and each link is known: URL fragment plus heading.
- **D4** The display formats are known: currency symbol, decimals, date format, how negative numbers are shown.
- **D5** The exact labels of every control the steps will use are known.

**E. Data and setup**
- **E1 [B]** A **verified** way to create each precondition exists: API, HTTP form POST or UI. It has been tried, not assumed.
- **E2** Each value is classified: **generated** (IDs, so capture it), **configurable** (settings, so capture it or pin it) or **fixed** (constant text).
- **E3** Isolation is possible: each case can own its data. It is known whether the environment is shared.
- **E4** Cleanup and reset constraints are known (for example no delete API, database resets).
- **E5** Test data restrictions are known: no real personal data, and nothing from holdout folders.

**F. Oracles**
- **F1 [B]** Every planned expected result has a source of truth **other than the page being checked**: an AC, a setup value, an invariant or another page.
- **F2** Invariants are identified: conservation (totals unchanged by internal transfers), uniqueness, how an item counts toward totals.
- **F3** For each oracle, record whether it is normative or descriptive. If the only source is descriptive, the cases are **characterization (regression) cases**. Flag this in the report, because they cannot catch bugs that already exist.

**G. Negative, edge and non-happy paths**
- **G1** Every rule has at least one violating condition listed, with its expected handling.
- **G2** Empty, zero, one and maximum states are covered or explicitly excluded (no accounts, zero balance, the maximum number of items).
- **G3** Session behaviours are marked in or out of scope: timeouts, back button, double submit, parallel sessions.

**H. Environment and known issues**
- **H1** The target URL and the build or version are known.
- **H2** Every configuration setting that affects expected values is known, along with how to read it (for example the ParaBank Admin "Init. Balance").
- **H3** Known quirks and bugs are listed **with the user's decision** for each (ignore, assert, or skip).
- **H4** The environment's stability is known (a shared public server, reset schedule, rate limits).

### 0.4 Tag every fact with its source
Every fact the cases will use goes into a **fact sheet**, and each fact carries exactly one tag:

| Tag | Meaning | Can support an expected result? |
|---|---|---|
| `[SRC-n AC-k]` | From a normative source | Yes (normative) |
| `[user YYYY-MM-DD]` | The user decided or confirmed it | Yes (normative) |
| `[observed YYYY-MM-DD]` | Seen in the running app or API response | Yes, but only as characterization; say so |
| `[screenshot SRC-n]` | Read from a screenshot | Yes for wording; not for correctness |
| `[setup]` | A value captured at runtime during setup | Yes |
| `[invariant]` | Follows logically from other tagged facts | Yes, if the facts it relies on are tagged |
| `[assumption]` | None of the above | **No.** It must become a question or an Open question |

### 0.5 Filling gaps, in this order
1. **Derive** it from the inputs you already have. Tag it `[invariant]` and name the facts it relies on.
2. **Explore** the running app or the API docs. Read, don't change: the only data you may create is your own test users. Never read holdout locations. Tag the result `[observed date]`.
3. **Ask** the user. Batch the questions and put the blocking ones first. Each question must be specific and answerable: "Is a $100.00 deposit allowed, or must it be more than $100?", not "What are the rules?". Don't ask anything that exploring could answer.
4. **Record it** as an Open question if the answer is still unknown. The cases that depend on it are marked **blocked**.

### 0.6 Verdict

| Verdict | Condition | Next step |
|---|---|---|
| **READY** | All [B] items PASS. No fact needed by an expected result is `[assumption]`. | Hand off to case writing |
| **READY WITH GAPS** | All [B] items PASS, but some non-blocking items FAIL | Hand off. The cases that touch those gaps are marked blocked or draft. The gaps are listed. |
| **NOT READY** | Any [B] item FAILS, or an expected result depends on an `[assumption]` | **Stop.** Return the questions. Do not generate cases. |

### 0.7 Output: the readiness report (the handoff contract)
```markdown
# Readiness report: <feature>  ·  Verdict: READY | READY WITH GAPS | NOT READY
## Sources          (table from 0.1)
## Checklist        (A1…H4: PASS/FAIL/N/A + tag + one-line note)
## Fact sheet
| F-ID | Fact | Tag | Used for (rule / planned check) |
## Test conditions  (one line per behaviour to test, each linked to F-IDs; no cases yet)
## Gaps and questions
| # | Checklist item | What's missing | Blocking? | Question for the user / resolution |
## Characterization warning (if F3 applies)
```
**Handoff rule:**
- The case-writing agent may use **only facts from the fact sheet**.
- Any expected result it needs that isn't in the sheet goes back as a gap. It must not be invented.
- Every expected result in the final cases must cite at least one F-ID, either in the case or in a traceability table.

### 0.8 Anti-patterns (automatic FAIL)
- Filling a gap with "typical" or industry-standard values (for example assuming a password needs 8 characters).
- Treating a screenshot as a complete spec, or treating observed behaviour as correct without flagging it.
- Quietly narrowing the scope so the checklist passes. Anything out of scope must be written down as out of scope.
- Asking the user something that a quick read-only exploration could answer.
- Rewording an acceptance criterion into something different and then testing the reworded version.
- Leaving an item marked PASS without a source tag.

---

## Required format
Each case has these fields, in this order:
1. **ID + Title:** `TC-NN: Verify that <actor/system> <behaviour> <condition>`
2. **Priority:** High / Medium / Low, plus one reason (risk, money movement, gate for other flows)
3. **Type:** Functional or E2E; UI or API
4. **Method:** which layer creates the data and which runs the steps. For example: "Setup through the API (S1–S3). Steps and verification through the UI only."
5. **Preconditions:** numbered, and each one names its **data-creation method** (a shared setup ID such as S1–S5, or an explicit endpoint)
6. **Test data:** a table. Runtime values are `{variables}`.
7. **Steps:** numbered, one user action per step, using the exact UI labels. Say which step captures which variable.
8. **Expected results:** numbered, and **every item starts with the step it checks**, for example "**After step 5:** …". An expected result without a step reference is not allowed.

## Rules
1. **One path per case.** No "if X then do Y" inside a case. Conditions become preconditions or separate cases.
2. **Independent cases.** Each case creates its own fresh user and data. No case depends on another case or on the order they run in.
3. **API is for data creation only.**
   - Preconditions use the fastest reliable layer: API, or HTTP form POST where no API exists.
   - In a UI test case, **every step is a UI action**: navigation, clicks, reading values. There are no API calls in Steps or Expected results.
   - Reach pages by clicking links and menus, not by typing URLs or calling the API. The click also checks that the link works.
   - Login through the UI only when login itself is being tested. Otherwise reuse the setup session.
4. **Every navigation click gets a check.** After clicking a link, check the destination: a URL fragment (for example `activity.htm?id={accountId}`) plus a heading or an identifying value.
5. **Runtime values are variables.** Never hard-code generated IDs, and never hard-code configurable amounts (for example the ParaBank starting balance). Capture them (`{initialAccountId}`, `{startBalance}`) and write expected values as formulas: `{startBalance} − 100.00`.
6. **Expected results are checkable as true or false.** Each one names an exact text, value, count, URL or state. Banned: "works", "is correct", "successfully", "displays properly".
7. **Use an independent oracle.** Don't check a page only against itself: a sum that matches the page's own total is not enough. Within the UI, also compare against:
   - a value you know from setup
   - a conserved invariant (internal transfers keep the total unchanged)
   - the same value shown on another page (overview vs details)
8. **Assert exact counts** ("exactly 3 rows") when the setup controls the data. This catches extra or missing items.
9. **Verify every place a change should appear.** For example, a new account should appear on the confirmation page, in Account Details, in the dropdowns, in the overview, and in the source account's balance.
10. **Record surprises** (API/UI mismatches, possible bugs) under "Open questions / findings". Don't silently work around them in the case.
11. **Checking UI against the API belongs to API or E2E cases.** Checking that the UI matches the backend is a separate concern. Write it as an API case or as an E2E reconciliation step, not inside a functional UI case.

## Review checklist
- Was there a readiness report, and is its verdict READY or READY WITH GAPS?
- Does every expected result cite an F-ID from the fact sheet?
- Is any expected result backed only by an observed fact without being flagged as characterization?
- Does every precondition name how its data is created?
- Are there any API calls in Steps?
- Does every expected result start with "After step N"?
- Does every link click have a destination check?
- Is anything hard-coded (IDs, amounts)?
- Are there any vague words?
- Is there a count or an independent oracle?

## ParaBank facts (verified 2026-09-26)
- There is no REST register endpoint. Register with `POST /parabank/register.htm` (form fields `customer.*` and `repeatedPassword`), after a `GET` that returns a `JSESSIONID`. The session from the POST is logged in, and it can be reused in the browser.
- `createAccount` `newAccountType` values: `0` = CHECKING, `1` = SAVINGS. Its response shows `balance: 0`. Ignore that and take expected balances from the UI (100.00).
- A session created by an HTTP client (API register) is not the browser's session. Hand it over (`page.request` or `context.addCookies`) before UI steps.
- A new user has 1 CHECKING account at the Admin "Init. Balance", currently $515.50. Opening an account moves $100.00.
- The account link goes to `activity.htm?id={accountId}` (heading `Account Details`).
- The UI shows dates as `MM-DD-YYYY`. The transaction names are `Funds Transfer Received` on the new account and `Funds Transfer Sent` on the source account.
