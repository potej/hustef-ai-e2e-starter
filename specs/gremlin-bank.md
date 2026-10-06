# Gremlin Bank: Authentication, Dashboard, and Domestic Transfers

## Application Overview

Explored on 2026-10-06 using seed.spec.ts, which signs in with GREMLIN_USER and GREMLIN_PASSWORD from .env. The browser displayed Release 1. Scope: sign in/out, dashboard accounts/recent transactions/chart data, and domestic HUF transfers. Exactly 20 independent scenarios follow; related values are grouped into matrices rather than separate scenarios.

## Starting State and Execution Rules
Every scenario starts with a new browser context and fresh bank state, with no prior transfers, drafts, or authenticated session. Unless a scenario explicitly tests login, run the sign-in steps from seed.spec.ts inside that scenario. Never reuse another scenario's cookies or receipt. Each matrix row requiring confirmation or a pristine daily allowance starts in its own fresh context; exceptions are the explicitly ordered cumulative-limit steps within scenario 17. Use GREMLIN_URL and the repository's GREMLIN_RELEASE configuration. These exact baseline values were observed in Release 1; if the release changes, re-explore before changing expectations. Do not silently weaken assertions.

Credentials are read from environment variables, never embedded in tests or this plan. The transaction PIN is published demo test data in tests/walls/support.ts (TEST_PIN), not a real banking credential. Use that documented value for valid-PIN steps and a different four-digit value for invalid-PIN steps. Normal tests import test and expect from tests/fixtures.ts. Use accessible roles, labels, and text with web-first assertions; do not use CSS/XPath, fixed sleeps, or networkidle. Wait for loaded account regions, IBAN verification status, approval controls, and final receipts.

## Baseline Business Data
Everyday Account: IBAN HU39 9992 0265 3141 5926 5358 9797; balance 1,250,000 HUF. Savings Account: IBAN HU03 9992 0265 2718 2818 2845 9043; balance 5,400,000 HUF.

Saved payees: Kiss Péter / HU72 9990 1017 1618 0339 8874 9892; Nagy Eszter / HU71 9990 2025 1414 2135 6237 3099; Tóth Bence / HU03 9990 3033 1732 0508 0756 8879. All are fictional Hungarian IBANs.

Initial recent transactions, newest first: 2026-09-30 | Grocery store, Budapest | -18,450 HUF; 2026-09-29 | Salary, Gremlin Works Ltd. | +685,000 HUF; 2026-09-27 | Mobile phone bill | -7,990 HUF; 2026-09-25 | Card payment, bookshop | -12,300 HUF; 2026-09-24 | Transfer from Savings Account | +50,000 HUF.

Observed chart fixture: 30 consecutive dates, oldest first, ending on the application date. On exploration the range was 2026-09-07 through 2026-10-06. Amounts by row in HUF: [12400, 0, 8350, 23100, 4500, 0, 15990, 9200, 31800, 2750, 0, 18400, 6650, 12000, 0, 27300, 4100, 9900, 14250, 0, 7600, 21450, 3300, 11800, 0, 16700, 5400, 19950, 8800, 13500]. Use the application's date/timezone for relative-date checks; do not freeze a future run to the exploration date.

Observed transfer rules: positive whole-HUF amounts; commas and spaces may group digits. Fee examples match max(200, round(amount * 0.003)) HUF; total = principal + fee. Do not assume a 5,000 HUF cap: 2,000,000 HUF has a 6,000 HUF fee. Funds validation includes fees. Per-transfer ceiling is 10,000,000 HUF; cumulative daily principal ceiling is 2,000,000 HUF across accounts, excluding fees. The daily limit and baseline funds prevent an end-to-end successful 10,000,000 HUF transfer, so test its validation precedence instead of inventing funded state.

## Approval and Testability
The visible Transaction PIN input says '4 digits' but lives inside a closed shadow root and is absent from accessibility snapshots. The repository documents a keyboard workaround: focus Confirm transfer, press Shift+Tab to enter the PIN control, clear any existing content, and type the PIN. Do not use script injection to set internal PIN state. A correct PIN opens the Confirm payment dialog; its Gremlin Secure titled iframe contains Approve payment and displays the payee, IBAN, and total debit. Cancel returns to review without committing. The IBAN control uses an open shadow root and is reachable by role.

Successful confirmation, receipt reload, dashboard reconciliation, invalid/blank PIN, approval cancellation, fee/funds/limit probes, and cumulative allowance across accounts were observed. Boundary tests for name/reference lengths, stale review navigation, and retry/double submission are planned regression checks; their complete outcomes were not executed during exploration. Review and receipt currently do not display the user-entered payment Reference; the receipt's generated GB- reference is a separate identifier. Do not confuse these or invent a visible reference field.

## Pass and Fail Criteria
A scenario passes only when every listed business value, validation state, and navigation outcome holds. Validation, review, cancellation, and failed authorization must not mutate balances, transaction history, or daily allowance. A wrong debit/fee/payee, missing or duplicate posting, unauthorized access, bypassed limit, stale IBAN verification, or unintended submission is a failure. Each scenario states additional failure conditions. Negative scenarios are expected to remain usable after correction.

## Test Scenarios

### 1. Authentication

**Seed:** `seed.spec.ts`

#### 1.1. 01. Valid credentials establish an authenticated session [high]

**File:** `tests/auth/sign-in.spec.ts`

**Steps:**
  1. Starting state: fresh signed-out context. Open /login using GREMLIN_URL.
    - expect: Heading 'Sign in to Gremlin Bank', Username and Password inputs, and Sign in button are visible. No signed-in identity or Sign out button is present. Password input masks entered characters.
  2. Fill Username with env('GREMLIN_USER') and Password with env('GREMLIN_PASSWORD'); click Sign in. Wait for Accounts and both loaded account regions.
    - expect: URL is /dashboard, heading is Accounts, signed-in identity matches GREMLIN_USER, and Sign out is available.
    - expect: Everyday balance is 1,250,000 HUF and Savings balance is 5,400,000 HUF.
  3. Reload /dashboard.
    - expect: Session remains authenticated and initial balances are unchanged. Failure: login rejection, lost session on reload, wrong identity, or mutated bank state.

#### 1.2. 02. Missing and incorrect credentials are rejected and correction succeeds [high]

**File:** `tests/auth/invalid-credentials.spec.ts`

**Steps:**
  1. Starting state: fresh signed-out context for each row. Open /login and submit the matrix: both fields empty; valid username with empty password; empty username with valid password; nonexistent-workshop-user with a nonempty invalid password; valid username with a wrong password.
    - expect: Each submission stays on /login and shows the generic alert 'Wrong username or password.'
    - expect: No signed-in identity or account data appears, and the message does not distinguish nonexistent users from wrong passwords.
  2. After a rejected attempt, enter the environment's valid credentials and click Sign in.
    - expect: Accounts loads successfully with initial balances. Failure: any invalid row authenticates, a rejection exposes account data, or correction cannot recover.

#### 1.3. 03. Sign out ends the session and permits a new sign in [high]

**File:** `tests/auth/sign-out.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in using seed.spec.ts steps and wait for loaded dashboard accounts. Click Sign out.
    - expect: Navigation ends at /login. Sign in controls replace the signed-in identity and Sign out control.
  2. Use browser Back, then reload any restored protected page; directly revisit /dashboard.
    - expect: A live protected view cannot be recovered without signing in; navigation/reload returns to /login.
    - expect: No new action or transfer can run using the signed-out session.
  3. Sign in again with the environment credentials.
    - expect: Accounts and original balances load. Failure: session remains usable after sign-out or valid re-login fails.

#### 1.4. 04. Protected bank routes reject unauthenticated access [high]

**File:** `tests/auth/protected-routes.spec.ts`

**Steps:**
  1. Starting state: separate fresh signed-out context for each path. Directly navigate to /dashboard, /transfer, /transfer/review, and /transfer/done.
    - expect: Each path returns to /login with sign-in controls; no account, review, or receipt data is disclosed. Dashboard, form, and review redirects were observed; the done route is an additional regression check.
  2. Sign in with valid environment credentials after a redirect.
    - expect: An authenticated dashboard is available with fresh balances. Failure: any protected route is accessible while signed out or unauthorized access changes bank state.

### 2. Dashboard

**Seed:** `seed.spec.ts`

#### 2.1. 05. Loaded accounts display correct names, IBANs, and balances [medium]

**File:** `tests/dashboard/accounts.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in with seed steps and wait until Loading accounts is replaced by the account regions.
    - expect: Exactly the Everyday Account and Savings Account regions are present; no loading text remains.
  2. Inspect each account region and compare its heading, IBAN, balance, and currency with Baseline Business Data.
    - expect: Everyday: HU39 9992 0265 3141 5926 5358 9797 and 1,250,000 HUF. Savings: HU03 9992 0265 2718 2818 2845 9043 and 5,400,000 HUF.
    - expect: Balances and IBANs are associated with the correct account, not asserted globally.
  3. Click New transfer, then Back to accounts; reload the dashboard and wait for loaded accounts.
    - expect: Navigation alone does not change either account balance or the initial transaction count. Failure: wrong account data, persistent loading, or navigation mutates balances.

#### 2.2. 06. Recent transactions preserve business values and ordering [medium]

**File:** `tests/dashboard/recent-transactions.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in and locate the Recent transactions table by its accessible name.
    - expect: Column headers are Date, Description, and Amount, with exactly five initial data rows.
  2. Compare each complete row against the five initial rows in Baseline Business Data.
    - expect: Every date, description, signed amount, and HUF currency matches its own row. Rows are newest first.
    - expect: Salary and savings transfer are positive credits; grocery, mobile, and bookshop are negative debits.
  3. Reload and inspect the table again.
    - expect: The same five initial transactions remain, without duplication or reordering. Failure: missing/extra row, swapped signs, wrong amount, or mismatched descriptions.

#### 2.3. 07. Chart data expands, exposes all 30 values, and collapses cleanly [low]

**File:** `tests/dashboard/chart-data.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in and locate Spending in the last 30 days.
    - expect: Show chart data is available and the chart-data table is initially hidden.
  2. Click Show chart data and inspect the accessible spending table.
    - expect: Control changes to Hide chart data. The table has Date and Amount columns and exactly 30 data rows.
    - expect: Dates are consecutive, oldest first, ending on the application's current date. On 2026-10-06 they span 2026-09-07 through 2026-10-06.
    - expect: All 30 HUF values match the chart fixture in Baseline Business Data, including the six zero-spend days; the first value is 12,400 HUF and last is 13,500 HUF.
  3. Click Hide chart data, then Show chart data again.
    - expect: The table disappears and returns with the same 30 rows, without duplicates. Accounts and recent transactions are unaffected. Failure: incorrect/missing chart values, omitted zeros, bad dates, or broken toggle.

### 3. Domestic HUF Transfer

**Seed:** `seed.spec.ts`

#### 3.1. 08. Account selection and saved payees populate the correct form data [high]

**File:** `tests/transfer/defaults-and-payees.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in and click New transfer.
    - expect: Form contains From account, Beneficiary name, IBAN, Check IBAN, Amount (HUF), Reference, Continue, and Back to accounts.
    - expect: Everyday Account is selected with Available: 1,250,000 HUF. Beneficiary, IBAN, amount, and reference are initially blank. The advertised limits are 10,000,000 HUF per transfer and 2,000,000 HUF per day.
  2. Select Savings Account, then Everyday Account.
    - expect: Available updates to 5,400,000 HUF and then 1,250,000 HUF, matching the selected account.
  3. For each of Kiss Péter, Nagy Eszter, and Tóth Bence, click its Use button and compare Beneficiary name and IBAN with Baseline Business Data.
    - expect: Both fields reflect the selected payee, replacing the prior payee rather than combining values. Amount, reference, and chosen account are not unexpectedly altered.
    - expect: Selecting a saved payee does not itself verify its IBAN; Check IBAN is still required.
  4. Return via Back to accounts without submitting.
    - expect: Initial balances and five recent transactions remain. Failure: incorrect payee mapping, incorrect Available amount, or state mutation.

#### 3.2. 09. Required transfer fields reject blank and whitespace-only values [high]

**File:** `tests/transfer/required-fields.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in, open New transfer, leave the form blank, and click Continue.
    - expect: Remain on the form with 'Enter a beneficiary name.', 'Check the IBAN first.', and 'Enter an amount greater than 0.' associated with the appropriate fields.
  2. With a valid verified payee IBAN and amount 1000, submit Beneficiary name as blank and then three spaces; use a fresh form or clear prior errors for each row.
    - expect: Both beneficiary values are rejected with 'Enter a beneficiary name.' No review page or debit is created.
  3. Correct the beneficiary, leave Reference blank, verify the IBAN, and Continue.
    - expect: Review accepts the optional blank Reference; amount is 1,000 HUF, fee 200 HUF, total 1,200 HUF.
    - expect: Failure: missing required values reach review, whitespace beneficiary is accepted, or optional reference is incorrectly required. No funds move until final approval.

#### 3.3. 10. Domestic IBAN checking validates checksum, country, and normalization [high]

**File:** `tests/transfer/iban-validation.spec.ts`

**Steps:**
  1. Starting state: fresh context per matrix row. Sign in, open New transfer, enter a beneficiary and amount 1000. Fill IBAN and click Check IBAN for: blank; HU72 9990; HU00 9990 1017 1618 0339 8874 9892; DE89 3704 0044 0532 0130 00.
    - expect: Each row shows Invalid IBAN and an invalid input state. A valid foreign IBAN is rejected for this domestic workflow.
  2. For each invalid row, click Continue.
    - expect: Remain on the form with the IBAN-check requirement; no review or debit is possible.
  3. Repeat valid rows with HU72 9990 1017 1618 0339 8874 9892 and lowercase compact hu72999010171618033988749892. Click Check IBAN, wait for IBAN verified status, then Continue.
    - expect: Both are accepted and normalized to the uppercase grouped Hungarian IBAN. Review has the same payee and normalized IBAN.
    - expect: Failure: invalid/foreign IBAN accepted, valid domestic formatting rejected, or normalization changes the account number.

#### 3.4. 11. Editing or replacing a verified IBAN requires a new check [high]

**File:** `tests/transfer/iban-reverification.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in, select Kiss Péter, enter amount 1000, Check IBAN, and wait for verification.
    - expect: The verified status corresponds to Kiss Péter's IBAN.
  2. Replace the IBAN with Tóth Bence's valid IBAN without clicking Check IBAN; click Continue.
    - expect: Verification is invalidated and Continue stays on the form with 'Check the IBAN first.' No stale verified state is reused.
  3. Check the new IBAN and Continue; inspect review. Return with Change details, use Nagy Eszter, and try Continue before checking again.
    - expect: Review uses the newly checked IBAN, not the old one. Replacing the payee must also require verification of the newly populated IBAN.
  4. Check Nagy Eszter's IBAN and Continue.
    - expect: Review now displays Nagy Eszter and the correct IBAN. Failure: a stale check authorizes a different IBAN or review retains the previous recipient.

#### 3.5. 12. Amount parsing rejects invalid values and accepts grouped whole HUF [high]

**File:** `tests/transfer/amount-validation.spec.ts`

**Steps:**
  1. Starting state: fresh context per row. Sign in, fill a valid beneficiary and checked domestic IBAN, leave Reference blank. Submit amounts: blank, three spaces, 0, -1, abc, 1000abc, 0.5, and 1.5.
    - expect: Every row stays on the form with 'Enter an amount greater than 0.' No fractional truncation, partial-number parsing, review, or debit occurs.
  2. Repeat valid rows with 1, 1000, 1,000, and 1 000; Continue and inspect review without approving.
    - expect: 1 is accepted as 1 HUF with fee 200 HUF and total 201 HUF.
    - expect: The other three formats all produce 1,000 HUF, fee 200 HUF, and total 1,200 HUF.
  3. Return to accounts after the probes.
    - expect: Both original balances and five initial transactions remain. Failure: silent conversion of malformed input, rejection of supported grouping, or validation/review moves funds.

#### 3.6. 13. Beneficiary and reference enforce their length boundaries [high]

**File:** `tests/transfer/text-boundaries.spec.ts`

**Steps:**
  1. Starting state: fresh context per row. Sign in and fill a verified domestic IBAN and amount 1000. Test Beneficiary name lengths 1, 70, and 71 characters using repeated ASCII letters; test Reference lengths 0, 1, 140, and 141 characters separately.
    - expect: Beneficiary input limit is 70; Reference input limit is 140. Values at their permitted boundaries remain usable.
    - expect: Typing or pasting an overlength value must not allow a stored value beyond the corresponding limit; the observed HTML maxlength may prevent extra characters. Do not expect an unobserved server error message.
  2. For each accepted row, Continue and inspect the beneficiary on review; return with Change details and inspect both form values.
    - expect: The permitted beneficiary value is preserved exactly and the reference is retained on the edit form. Reference remains optional.
    - expect: Current review/receipt does not expose the user-entered reference; do not assert a nonexistent row or confuse it with the generated receipt ID.
  3. Leave without approval and verify accounts.
    - expect: No debit or transaction is created. Failure: accepted values are corrupted/lost, overlength content exceeds limits, or valid boundary values cannot reach review. Full boundary outcomes require execution.

#### 3.7. 14. Fee minimum, percentage, rounding, and large amounts calculate correctly [high]

**File:** `tests/transfer/fees.spec.ts`

**Steps:**
  1. Starting state: fresh context per row. Sign in, select Savings Account for all rows, use and verify Kiss Péter, and Continue for the matrix amount / fee / total in HUF: 1 / 200 / 201; 10,000 / 200 / 10,200; 50,000 / 200 / 50,200; 66,666 / 200 / 66,866; 66,667 / 200 / 66,867; 66,833 / 200 / 67,033; 66,834 / 201 / 67,035; 100,000 / 300 / 100,300; 100,001 / 300 / 100,301; 1,000,000 / 3,000 / 1,003,000; 1,666,666 / 5,000 / 1,671,666; 1,666,667 / 5,000 / 1,671,667; 1,999,999 / 6,000 / 2,005,999; 2,000,000 / 6,000 / 2,006,000.
    - expect: Every review row shows its exact principal, fee, and total in HUF. Total always equals amount plus fee.
    - expect: Examples distinguish the 200 HUF minimum, nearest-whole-HUF rounding transition, percentage fee, and absence of a presumed 5,000 HUF cap.
    - expect: A 2,000,000 HUF principal is allowed although its fee-inclusive total exceeds 2,000,000 HUF.
  2. Do not confirm any row; leave review and inspect the selected account balance.
    - expect: Savings remains at 5,400,000 HUF. Failure: incorrect fee/rounding/total or review consumes balance or daily allowance.

#### 3.8. 15. Funds validation includes the fee and accepts an exact-balance debit [high]

**File:** `tests/transfer/insufficient-funds.spec.ts`

**Steps:**
  1. Starting state: fresh context per row. Sign in, use Everyday Account and a verified Kiss Péter IBAN. Test amounts 1,246,261; 1,246,262; and 1,250,000 HUF.
    - expect: 1,246,261 reaches review with fee 3,739 and total 1,250,000 HUF, exactly the available balance.
    - expect: 1,246,262 and 1,250,000 stay on the form with Insufficient funds. Principal alone fitting the balance is not sufficient.
  2. For the exact-balance accepted row only, enter the valid demo PIN, Confirm transfer, verify the approval iframe's total, and Approve payment.
    - expect: Receipt shows principal 1,246,261 HUF, fee 3,739 HUF, total 1,250,000 HUF, and new Everyday balance 0 HUF.
    - expect: Dashboard agrees; Savings remains 5,400,000 HUF. Zero balance must not be rejected or become negative.
  3. For a rejected row in a separate fresh context, switch to Savings Account and resubmit the same verified-payee amount.
    - expect: The amount is fundable from Savings and review identifies Savings. The failed Everyday attempt has not consumed allowance or posted a transaction.
    - expect: Failure: funds check excludes fees, exact available total is rejected, negative balance is created, or the wrong source is debited.

#### 3.9. 16. Daily and per-transfer limits enforce their adjacent boundaries [high]

**File:** `tests/transfer/limit-boundaries.spec.ts`

**Steps:**
  1. Starting state: fresh context per matrix row with no transfers today. Sign in, choose Savings Account, and fill a verified payee. Submit principals 1,999,999; 2,000,000; and 2,000,001 HUF.
    - expect: 1,999,999 and 2,000,000 reach review with totals 2,005,999 and 2,006,000 HUF respectively.
    - expect: 2,000,001 is blocked with 'Daily limit of 2,000,000 HUF exceeded.' Fees are excluded from the daily-principal ceiling.
  2. In separate fresh contexts submit 9,999,999; 10,000,000; and 10,000,001 HUF.
    - expect: 9,999,999 and 10,000,000 are blocked by the daily limit, not reported as exceeding the single-transfer maximum.
    - expect: 10,000,001 is blocked with 'The maximum single transfer is 10,000,000 HUF.' The single-transfer error takes precedence over daily/funds errors.
    - expect: The exact single-transfer ceiling cannot successfully complete under the lower daily limit and baseline balances; do not fabricate an accepted 10,000,000 HUF payment.
  3. Return to accounts without approval after each rejection/review.
    - expect: No debit or transaction was created. Failure: one-HUF-over boundary accepted, inclusive ceiling misclassified, or fees incorrectly counted as daily principal.

#### 3.10. 17. Daily principal allowance accumulates across accounts and resets only with fresh bank state [high]

**File:** `tests/transfer/cumulative-daily-limit.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in and complete a 100,000 HUF Everyday-to-Kiss Péter payment with valid PIN and iframe approval.
    - expect: Everyday becomes 1,149,700 HUF after the 300 HUF fee; Savings stays 5,400,000 HUF. Today's remaining principal allowance is 1,900,000 HUF.
  2. In the same context start a Savings-to-Nagy Eszter transfer for 1,900,001 HUF with checked IBAN.
    - expect: Blocked by Daily limit of 2,000,000 HUF exceeded despite sufficient Savings funds. Switching source accounts cannot reset the allowance.
  3. Correct the amount to 1,900,000 HUF and Continue. Check fee 5,700 and total 1,905,700; complete valid PIN and approval.
    - expect: The exact remaining principal allowance is accepted. Savings becomes 3,494,300 HUF; Everyday remains 1,149,700 HUF. Daily principal is exactly 2,000,000 HUF.
  4. In the same context try a further 1 HUF transfer from either account. Sign out, sign in again, and retry that 1 HUF transfer.
    - expect: Both attempts remain blocked by the daily limit. Re-authentication does not reset the cookie-held bank ledger.
    - expect: Only the two approved payments are posted; the rejected attempts consume no funds.
  5. Open a separate fresh context, sign in, and review a Savings transfer for 2,000,000 HUF without approving.
    - expect: Original balances and a fresh daily allowance are restored by isolation, not by re-login. Failure: per-account allowance bypass, incorrect inclusion of fees, re-login resets allowance, or fresh contexts inherit prior transfers.

#### 3.11. 18. Review is nonmutating and edited details replace the draft [high]

**File:** `tests/transfer/review-and-edit.spec.ts`

**Steps:**
  1. Starting state: fresh context. Sign in, choose Everyday Account, use Kiss Péter, check IBAN, enter amount 100000 and Reference 'Domestic review', and Continue.
    - expect: Review Transfer details shows From Everyday Account, To Kiss Péter, correct normalized IBAN, amount 100,000 HUF, fee 300 HUF, total 100,300 HUF.
    - expect: Confirm transfer and Change details are available; no balance debit occurs merely on review.
  2. Click Change details and inspect From account, Beneficiary name, IBAN, Amount, and Reference.
    - expect: Draft values are preserved, including the user-entered Reference. The reference is not a review-table field in the explored release.
  3. Change source to Savings, select Nagy Eszter, explicitly Check IBAN, change amount to 50000 and Reference to 'Edited domestic review', and Continue.
    - expect: Review reflects Savings, Nagy Eszter and her IBAN, amount 50,000 HUF, fee 200 HUF, total 50,200 HUF; no old recipient/source/amount remains.
  4. Before approval, navigate to /dashboard and then attempt a direct /transfer/review in the same context; separately check /transfer/review with no draft in a newly signed-in context.
    - expect: Balances and history remain initial. A missing draft must safely return to the form or show a clear no-draft state, not an unrelated/stale transfer or allow approval. Exact no-draft navigation is a planned regression check, not an observed message.
    - expect: Failure: lost edit data, stale review calculations, incorrect payee, debit before approval, or an approvable absent/unrelated draft.

#### 3.12. 19. Failed PIN and canceled payment approval never submit a transfer [high]

**File:** `tests/transfer/authorization-and-cancel.spec.ts`

**Steps:**
  1. Starting state: fresh context per PIN row. Sign in and review an Everyday-to-Kiss Péter transfer for 100000 HUF. Using keyboard focus into Transaction PIN, submit with blank PIN, a wrong four-digit PIN, and an incomplete one-to-three-digit PIN.
    - expect: No receipt is produced and no debit occurs. Blank/wrong PIN displays Wrong PIN; incomplete input must not approve a payment. Remain on review or show a clear input-validation state.
    - expect: Do not require a locator inside the closed shadow root; use the documented Confirm-button focus, Shift+Tab, clear, and keyboard typing sequence.
  2. Correct the PIN with the documented valid demo value and click Confirm transfer.
    - expect: Confirm payment dialog appears. In its Gremlin Secure iframe, the heading requests approval of 100,300 HUF and the payee/IBAN match review.
    - expect: PIN acceptance alone must not commit the payment.
  3. Click Cancel without Approve payment; inspect review and navigate to accounts.
    - expect: Dialog closes and review remains usable. Accounts retain 1,250,000 and 5,400,000 HUF and the original five transactions.
    - expect: No allowance is consumed; in the same canceled-payment context a Savings principal of 2,000,000 HUF can still reach review.
  4. In a separate fresh context, perform a failed-PIN attempt then correct and fully approve the original payment.
    - expect: Correction can succeed exactly once. Failure: wrong/partial PIN accepted, cancel posts a payment, approval amount/payee differs from review, or failed authorization prevents valid recovery.

#### 3.13. 20. Approved payment produces one consistent receipt, debit, and transaction [high]

**File:** `tests/transfer/confirmation-and-reconciliation.spec.ts`

**Steps:**
  1. Starting state: fresh context per source-account row. Sign in and make a 100000 HUF domestic payment to Kiss Péter with checked IBAN and a descriptive Reference; run once from Everyday and once from Savings.
    - expect: Review amount is 100,000 HUF, fee 300 HUF, total 100,300 HUF, with the selected source and correct payee/IBAN.
  2. Enter the valid demo PIN using keyboard navigation, click Confirm transfer, verify the iframe displays 100,300 HUF and the correct payee/IBAN, then click Approve payment.
    - expect: Transfer submitted receipt appears at /transfer/done.
    - expect: Receipt shows Paid to Kiss Péter, correct IBAN, amount 100,000 HUF, fee 300 HUF, total 100,300 HUF, and PIN check passed.
    - expect: Generated receipt Reference matches the observed format GB- followed by six uppercase alphanumeric characters; do not assert the random identifier itself.
    - expect: Everyday-source row: new Everyday balance 1,149,700 HUF and Savings unchanged at 5,400,000 HUF. Savings-source row: new Savings balance 5,299,700 HUF and Everyday unchanged at 1,250,000 HUF.
  3. Reload the receipt, click Back to accounts, wait for loaded accounts, and inspect Recent transactions.
    - expect: Receipt reload preserves the same generated reference and does not repeat the debit.
    - expect: Selected account agrees with receipt; other account is unchanged. Exactly one new newest row reads Transfer to Kiss Péter with -100,300 HUF on the application's transfer date; the five prior rows remain intact.
    - expect: Debit row includes the fee, not just the 100,000 HUF principal.
  4. Reload the dashboard and use browser Back/Forward to revisit confirmation history without creating a new transfer.
    - expect: Exactly one debit and one transaction remain. A consumed draft cannot authorize a second payment; any deliberate rapid repeated approval interaction must also post at most once.
    - expect: Failure: inconsistent receipt/review/dashboard values, wrong source debit, random-reference literal assertion, duplicate debit, missing transaction, or changed baseline rows. Rapid repeated approval and stale-history behavior are planned regression checks.
