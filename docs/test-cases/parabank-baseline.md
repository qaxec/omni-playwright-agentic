# ParaBank baseline test cases

Status: **baseline scope** (agreed 2026-09-26). These 3 cases are the starting scope for implementation.
Rules for writing cases: the `test-case-writing` skill ([.claude/skills/test-case-writing/SKILL.md](../../.claude/skills/test-case-writing/SKILL.md)).

- UI: `https://parabank.parasoft.com/parabank/index.htm`
- REST base: `https://parabank.parasoft.com/parabank/services/bank` (send `Accept: application/json`)

## Shared setup methods

Every case creates its own fresh user, so each case runs independently and in any order.

**API is for data creation only.** S1–S5 are used only in **Preconditions**. Every test step (navigation, clicks, reading values) goes through the UI. This way each click also checks that its link or button works.

| ID | Method | How | Returns |
|---|---|---|---|
| **S1 Register user** | HTTP form POST (ParaBank has **no REST register endpoint**) | 1. `GET /parabank/register.htm` to get a `JSESSIONID` cookie. 2. `POST /parabank/register.htm` (form-urlencoded) with `customer.firstName`, `customer.lastName`, `customer.address.street`, `customer.address.city`, `customer.address.state`, `customer.address.zipCode`, `customer.phoneNumber`, `customer.ssn`, `customer.username`, `customer.password`, `repeatedPassword`. The response page contains `Your account was created successfully.` | A registered user. That HTTP session is logged in. |
| **S2 Get customer** | REST | `GET /services/bank/login/{username}/{password}` | `{customerId}` (field `id`) |
| **S3 Get accounts** | REST | `GET /services/bank/customers/{customerId}/accounts` | For a new user: exactly 1 CHECKING account, giving `{initialAccountId}` and `{startBalance}` |
| **S4 Open account** | REST | `POST /services/bank/createAccount?customerId={customerId}&newAccountType={0=CHECKING, 1=SAVINGS}&fromAccountId={id}` | New account `id`. Ignore `balance` in this response: it always returns `0`. The expected balance is what the UI shows, `100.00`. |
| **S5 UI session** | Cookie reuse | S1 logs in the **HTTP client**, not the browser. They are two separate clients, each with its own cookies. S5 gives the S1 `JSESSIONID` cookie to the browser. In Playwright, run S1 with `page.request` (the browser context's own client) so the cookie is shared automatically; otherwise copy it with `context.addCookies()`. | A browser that is already logged in, without going through the UI login. Without S5 the browser opens on the Customer Login page. |

**Default user profile (test data for S1):**

| Field | Value |
|---|---|
| First / Last name | Mary / John |
| Address / City / State / Zip | 123 North / West Fargo / ND / 58078 |
| Phone / SSN | 218123456 / 123456789 |
| Username | `qa_<yyyyMMddHHmmss><rand>` (unique per test) |
| Password | `123456` |

**Captured variables:** `{username}`, `{customerId}`, `{initialAccountId}`, `{startBalance}`.
- **Never hard-code account IDs.** They change on every registration.
- **Never hard-code the $515.50 starting balance.** It is the Admin Page "Init. Balance" setting and can be changed. Use `{startBalance}` instead.

Observed on 2026-09-26: the new-user account is CHECKING with $515.50. Opening an account moves $100.00, the minimum deposit.

---

## TC-01: Verify that a registered user can log in through the UI with valid credentials

| | |
|---|---|
| **Priority** | High (every UI flow depends on login) |
| **Type** | Functional, UI |
| **Method** | Setup through the API (S1–S3). Steps and verification through the **UI** only. |

**Preconditions**
1. S1: register a fresh `{username}` / `123456` through the HTTP POST. **Do not** reuse this session in the browser, so the browser stays logged out.
2. S2 and S3: capture `{customerId}`, `{initialAccountId}` and `{startBalance}`.
3. The browser has no ParaBank session.

**Test data**

| Field | Value |
|---|---|
| Username | `{username}` |
| Password | `123456` |

**Steps**
1. Open `https://parabank.parasoft.com/parabank/index.htm`.
2. In **Customer Login**, enter `{username}` in **Username**.
3. Enter `123456` in **Password**.
4. Click **LOG IN**.

**Expected results**
1. The URL contains `/parabank/overview.htm`.
2. The page heading is `Accounts Overview`.
3. The left panel shows `Welcome Mary John`.
4. The **Log Out** link is visible under **Account Services**.
5. The accounts table has exactly 1 account row. Its account is `{initialAccountId}`, and its Balance and Available Amount both equal `{startBalance}`, formatted as `$515.50`.
6. The table's `Total` equals `{startBalance}`.

---

## TC-02: Verify that opening a CHECKING account funded from an existing account creates it and moves $100.00

| | |
|---|---|
| **Priority** | High (money movement, account creation) |
| **Type** | Functional, UI |
| **Method** | Setup through the API (S1–S3 and S5). Steps and verification through the **UI** only. |

**Preconditions**
1. S1: register a fresh user.
2. S2 and S3: capture `{customerId}`, `{initialAccountId}` and `{startBalance}`. S3 must return exactly 1 account.
3. S5: the browser is logged in as `{username}`.

**Test data**

| Field | Value |
|---|---|
| Account type | `CHECKING` |
| From account | `{initialAccountId}` |
| Transfer amount (system rule) | `100.00` |

**Steps**
1. In **Account Services**, click **Open New Account**.
2. In "What type of Account would you like to open?", select `CHECKING`.
3. In the "from account" dropdown, select `{initialAccountId}`.
4. Click **OPEN NEW ACCOUNT**. Capture the number shown after "Your new account number:" as `{newAccountId}`.
5. Click the `{newAccountId}` link.
6. Click **Open New Account** and read the options of the "from account" dropdown.
7. Click **Accounts Overview**.

**Expected results**
1. **After step 4:**
   - the heading is `Account Opened!`
   - the text `Congratulations, your account is now open.` is shown
   - `{newAccountId}` is numeric and different from `{initialAccountId}`
2. **After step 5, Account Details shows:**
   - the URL contains `activity.htm?id={newAccountId}` and the heading is `Account Details` (the link works)
   - Account Number = `{newAccountId}`
   - Account Type = `CHECKING`
   - Balance = `$100.00`
   - Available = `$100.00`
3. **After step 5, Account Activity has exactly 1 row:**
   - Date = today (`MM-DD-YYYY`)
   - Transaction = `Funds Transfer Received`
   - Debit is empty
   - Credit = `$100.00`
4. **After step 6:** the dropdown has exactly 2 options, `{initialAccountId}` and `{newAccountId}`.
5. **After step 7:**
   - `{initialAccountId}` Balance = `{startBalance} − 100.00`
   - `{newAccountId}` Balance = `$100.00`
   - Total = `{startBalance}`

---

## TC-03: Verify that Accounts Overview shows the correct balance for each account and the correct total

| | |
|---|---|
| **Priority** | Medium (read-only view of financial data) |
| **Type** | Functional, UI |
| **Method** | Setup through the API (S1–S5). Steps and verification through the **UI** only. |

**Preconditions**
1. S1: register a fresh user.
2. S2 and S3: capture `{customerId}`, `{initialAccountId}` and `{startBalance}`.
3. S4: open a CHECKING account (`newAccountType=0`) from `{initialAccountId}`, and save its ID as `{acc2}`.
4. S4: open a SAVINGS account (`newAccountType=1`) from `{initialAccountId}`, and save its ID as `{acc3}`.
5. S5: the browser is logged in as `{username}`.

**Test data (expected state after setup)**

| Account | Type | Balance | Available |
|---|---|---|---|
| `{initialAccountId}` | CHECKING | `{startBalance} − 200.00` | same as Balance |
| `{acc2}` | CHECKING | `100.00` | `100.00` |
| `{acc3}` | SAVINGS | `100.00` | `100.00` |
| **Total** | | **`{startBalance}`** | |

**Steps**
1. In **Account Services**, click **Accounts Overview**.
2. Read the Account, Balance and Available Amount of every row, and the `Total`.
3. Click the `{initialAccountId}` link. Read Account Type, Balance and Available on **Account Details**.
4. Click **Accounts Overview**, then click the `{acc2}` link and read the same fields.
5. Click **Accounts Overview**, then click the `{acc3}` link and read the same fields.

**Expected results**
1. **After step 1:** the heading is `Accounts Overview`, and the footnote `*Balance includes deposits that may be subject to holds` is shown.
2. **After step 2:**
   - the table has exactly 3 account rows: `{initialAccountId}`, `{acc2}` and `{acc3}`
   - each row's Balance and Available Amount equal the test-data table, shown as `$` with 2 decimals
   - `Total` = `{startBalance}`. It equals the sum of the 3 Balance values shown, and it is unchanged from the start because internal transfers do not change the total.
3. **After each of steps 3, 4 and 5:**
   - the URL contains `activity.htm?id={accountId}` and the heading is `Account Details` (the link works)
   - Account Number = that `{accountId}`
   - Account Type equals the test-data table
   - Balance and Available equal the values read on the overview in step 2

---

## Open questions / findings
- **Known quirk, not filed:** `POST /createAccount` returns `"balance": 0`. Decision: ignore the response balance. Expected balances come from the UI, which shows `100.00`.
- **Reserved for later:** a negative login case. Wrong password → the heading `Error!` and the message `The username and password could not be verified.`, and the user stays logged out. It is kept out of the baseline on purpose, to be used later as an input for verifying the AI workflow. Don't write or automate it by hand.
