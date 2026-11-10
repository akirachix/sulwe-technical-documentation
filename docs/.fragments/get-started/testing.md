---
title: Testing and QA
description: How Umucyo is tested — automated checks, the test case register, and ten worked examples.
---

# :material-test-tube: Testing and QA

<div class="at-a-glance" markdown>

**In short:** run the backend tests and the frontend checks before every pull request. Then try the change by hand in Swagger or Postman.

**Current state:** the backend suite passes with **637 tests**.

**Test case register:** every case the team has written lives in [:material-table-large: Umucyo Test Cases](https://docs.google.com/spreadsheets/d/1ohqQuJDhxkFJYOhb0hr6PcC6P6mhbrpQqFtdShplZ2U/edit?usp=sharing). Ten worked examples from it are on this page.

</div>

## How testing is organised

Umucyo is tested at three levels, and each one catches a different kind of mistake.

| Level | What it checks | Where it lives | Who runs it |
| --- | --- | --- | --- |
| **Automatic tests** | A single rule behaves as written — a refusal, a calculation, a rollback | `backend/tests/` (pytest) | CI, on every push |
| **Manual API testing** | A full flow works end to end against a real database | Swagger or the Postman collection | The person making the change |
| **Manual app testing** | The screen does what a person expects, on a phone if it is the agent app | A browser, a phone, or a preview deployment | The person making the change |

Automatic tests are the ones that keep working after you have forgotten why a rule exists. Manual tests are the ones that catch "it returns `200` but the button does nothing".

## The test case register

The authoritative list of test cases is the spreadsheet:

!!! note "Where the test cases live"
    [:material-table-large: Umucyo — Test Cases](https://docs.google.com/spreadsheets/d/1ohqQuJDhxkFJYOhb0hr6PcC6P6mhbrpQqFtdShplZ2U/edit?usp=sharing)

    When this page and the spreadsheet disagree, the spreadsheet wins — it is the record the team maintains. This page explains *how* to write a case, not which cases exist.

### The columns every case needs

A case that cannot be re-run by someone else is not a test case, it is an anecdote. Each row in the register carries these fields.

| Field | What goes in it | Example |
| --- | --- | --- |
| **ID** | Stable, never reused | `TC-014` |
| **Area** | Which feature or route | `Payments` |
| **Title** | One sentence, states the rule | `Sending the same payment twice pays once` |
| **Preconditions** | State of the system before the steps | `Signed in as admin of Coop A; one batch previewed` |
| **Steps** | Numbered, in order, no ambiguity | `1. POST /payments/{id}/send  2. repeat the same call` |
| **Expected result** | What the system must do, including the status code | `First 200; second 409; one payout record` |
| **Type** | Normal, boundary, invalid, permission, or repeat | `Repeat` |
| **Automated?** | `Yes` with the test name, or `No` with a reason | `Yes — tests/test_payments.py::test_send_is_idempotent` |
| **Status** | Pass, fail, or blocked | `Pass` |

### The five kinds of case

Every feature needs all five kinds, not just the happy one. This is the same checklist as further down the page, but as a naming scheme.

| Kind | Question it answers | Example |
| --- | --- | --- |
| **Normal** | Does the ordinary use work? | An agent records a delivery |
| **Boundary** | What happens at the exact edge of the allowed range? | A delivery of exactly 0 kg |
| **Invalid** | Is bad input refused cleanly? | A weight sent as text |
| **Permission** | Can the wrong person do it? | An agent tries to send a payment |
| **Repeat** | What happens if it is done twice? | The same delivery posted twice |

## Ten worked examples

These ten come from the register and cover all five kinds. Each row is written the way it should appear in the spreadsheet.

### 1. Normal — a delivery is recorded and priced

| Field | Content |
| --- | --- |
| **ID** | `TC-001` |
| **Area** | Deliveries |
| **Preconditions** | Signed in as an agent of Kigali Point; farmer `F-102` exists at that point; a price period covers today |
| **Steps** | 1. `POST /deliveries` with farmer `F-102`, gross weight 42.5 kg, tare 1.5 kg. 2. Read the response. 3. `GET /farmers/F-102/deliveries` |
| **Expected** | `201`; `net_weight` is `41.0`; `amount_owed` equals 41.0 × the period price; the record appears in the farmer's list |
| **Type** | Normal |
| **Automated** | Yes — `tests/test_deliveries.py::test_create_delivery_calculates_net_and_amount` |

The thing being proven here is the arithmetic, not the `201`. A delivery that saves but computes `amount_owed` from the gross weight passes a naive check and underpays or overpays every farmer.

### 2. Boundary — a delivery of zero net weight

| Field | Content |
| --- | --- |
| **ID** | `TC-002` |
| **Area** | Deliveries |
| **Preconditions** | Signed in as an agent with a farmer at their point |
| **Steps** | 1. `POST /deliveries` with gross 20.0 kg and tare 20.0 kg. 2. Repeat with gross 20.0 and tare 20.1 |
| **Expected** | First returns `400` (or the documented minimum) with a message naming weight; second is refused the same way. Nothing is written to the database |
| **Type** | Boundary |
| **Automated** | Yes — `tests/test_deliveries.py::test_zero_and_negative_net_weight_rejected` |

The second step matters more than the first: a tare greater than the gross produces a negative amount owed. If that is ever accepted, the cooperative owes money for tea it never received.

### 3. Invalid — a weight sent as text

| Field | Content |
| --- | --- |
| **ID** | `TC-003` |
| **Area** | Deliveries |
| **Preconditions** | Signed in as an agent |
| **Steps** | 1. `POST /deliveries` with `"gross_weight": "forty two"`. 2. Repeat with the field left out entirely |
| **Expected** | `422` both times, with the field named in the error. No record created, no `500` |
| **Type** | Invalid |
| **Automated** | Yes — `tests/test_deliveries.py::test_invalid_payload_returns_422` |

A `500` here would mean validation is happening in the wrong layer. Bad input must never reach the database.

### 4. Invalid — a phone number that cannot receive SMS

| Field | Content |
| --- | --- |
| **ID** | `TC-004` |
| **Area** | Farmers |
| **Preconditions** | Signed in as an agent |
| **Steps** | 1. `POST /farmers` with phone `"0788"`. 2. Repeat with `"+250788123456"`. 3. Repeat with `"0788123456"` |
| **Expected** | Steps 1 and 2 refused with a message showing the accepted format; step 3 accepted (`201`) |
| **Type** | Invalid |
| **Automated** | Yes — `tests/test_farmers.py::test_phone_normalisation` |

Worth writing down because the accepted format is the difference between a farmer getting an SMS and a farmer not getting paid. Whichever format the code accepts, this case records it permanently.

### 5. Permission — an agent tries to send a payment

| Field | Content |
| --- | --- |
| **ID** | `TC-005` |
| **Area** | Payments |
| **Preconditions** | Signed in as an **agent**; a batch exists for their point |
| **Steps** | 1. `POST /payments/{batch_id}/send` with the agent's token. 2. Repeat with no token at all |
| **Expected** | `403` with step 1, `401` with step 2. No payout record, no request to Safaricom |
| **Type** | Permission |
| **Automated** | Yes — `tests/test_payments.py::test_agent_cannot_send_payment` |

The second assertion is the one people forget: an unknown user must be stopped earlier than a known user with the wrong role, and the codes say which check ran.

### 6. Permission — another cooperative's farmer is invisible

| Field | Content |
| --- | --- |
| **ID** | `TC-006` |
| **Area** | Farmers |
| **Preconditions** | Farmer `F-900` belongs to Coop B; signed in as an administrator of Coop A |
| **Steps** | 1. `GET /farmers/F-900`. 2. `GET /farmers` and search for `F-900` |
| **Expected** | `404` on step 1 — not `403`. `F-900` absent from the list on step 2 |
| **Type** | Permission |
| **Automated** | Yes — `tests/test_tenancy.py::test_cross_cooperative_returns_404` |

`404` rather than `403` is deliberate: `403` would confirm the record exists. This case is what stops a future refactor from quietly changing one to the other.

### 7. Repeat — the same delivery posted twice

| Field | Content |
| --- | --- |
| **ID** | `TC-007` |
| **Area** | Deliveries |
| **Preconditions** | Signed in as an agent; one delivery already created with idempotency key `abc123` |
| **Steps** | 1. `POST /deliveries` again with the same key and the same body. 2. Repeat with the same key and a *different* weight |
| **Expected** | Step 1 returns the original record (`200`), not a second one — total deliveries stays at one. Step 2 returns `409` |
| **Type** | Repeat |
| **Automated** | Yes — `tests/test_deliveries.py::test_idempotency_key_reused` |

This is the case that makes retrying after a timeout safe. Without it, a flaky connection pays a farmer twice, so it earns its place at the top of the register.

### 8. Repeat — the same payment sent twice

| Field | Content |
| --- | --- |
| **ID** | `TC-008` |
| **Area** | Payments |
| **Preconditions** | Signed in as an admin; one batch sent successfully |
| **Steps** | 1. `POST /payments/{batch_id}/send` a second time. 2. Check the payout table and the mock Safaricom log |
| **Expected** | `409`, and exactly one outbound payout request in the log — no second charge |
| **Type** | Repeat |
| **Automated** | Yes — `tests/test_payments.py::test_send_is_idempotent` |

The log check is the important half. A `409` that still called the payment provider has already moved real money; the response code alone proves nothing.

### 9. Invalid — a payment sent while production payouts are off

| Field | Content |
| --- | --- |
| **ID** | `TC-009` |
| **Area** | Payments |
| **Preconditions** | Server setting for real payouts is off (the default) |
| **Steps** | 1. `POST /payments/{batch_id}/send` with the production flag in the request. 2. Repeat in sandbox mode with only the request flag set |
| **Expected** | Both refused (`400` or `409`) with a message explaining that both the server setting and the request flag are needed. No outbound call |
| **Type** | Invalid |
| **Automated** | Yes — `tests/test_payments.py::test_production_requires_server_setting` |

Written as a test case so that "how do we turn on real money?" has an answer in the repository, not in somebody's memory.

### 10. Normal — sign-in with the emailed code

| Field | Content |
| --- | --- |
| **ID** | `TC-010` |
| **Area** | Authentication |
| **Preconditions** | An administrator account exists; login codes are captured by the test harness rather than emailed |
| **Steps** | 1. `POST /auth/login` with the correct password. 2. `POST /auth/verify-otp` with the captured code. 3. Use the returned token on a protected route. 4. Repeat step 2 with the same code |
| **Expected** | Step 1 `200` with no token; step 2 `200` with an access token; step 3 `200`; step 4 refused — the code is spent |
| **Type** | Normal |
| **Automated** | Yes — `tests/test_auth.py::test_login_and_verify_otp` |

Step 4 is what makes the two-step login worth having. A code that can be used twice is a password with extra steps.

### Reading the ten together

| Kind | Cases | What the set proves |
| --- | --- | --- |
| Normal | 1, 10 | The system does its job |
| Boundary | 2 | The edge of the allowed range is guarded |
| Invalid | 3, 4, 9 | Bad input is refused before it reaches the database |
| Permission | 5, 6 | The wrong person is stopped, and cannot tell what exists |
| Repeat | 7, 8 | A retry is safe and does not move money twice |

## Automatic checks

=== "Backend"

    ```bash
    uv run pytest -q                 # all tests
    uv run ruff check .              # lint
    uv run ruff format --check .     # formatting
    uv run alembic heads             # exactly one migration head
    ```

    Tests run on an in-memory SQLite database, and login codes are captured instead of being emailed.

=== "Web apps"

    ```bash
    npm run lint          # ESLint
    npx tsc --noEmit      # type check
    npm run build         # production build
    ```

    A pre-commit hook also runs ESLint and Prettier on the files being committed.

### Running one test while you work

```bash
uv run pytest tests/test_payments.py::test_send_is_idempotent -q     # one test
uv run pytest tests/test_payments.py -q                             # one file
uv run pytest -k "idempotent or tenancy" -q                         # by name
uv run pytest -x --lf                                              # stop at the first failure, rerun last failures
```

`--lf` ("last failed") is the one worth keeping in your fingers: after a red build it runs only what broke, so the loop stays seconds long instead of a minute.

### What the suite does and does not cover

| Covered well | Not covered — check by hand |
| --- | --- |
| Route rules: status codes, roles, tenancy | How the screen looks and behaves on a real phone |
| Weight and money arithmetic | Whether an SMS actually arrives (sandbox logs only) |
| Database constraints and migrations | Timing under a slow network |
| Idempotency and repeat submissions | Browser storage, installed-PWA behaviour |

An automatic test proves the backend agreed with itself. It cannot prove the agent standing in the rain saw the right thing.

## Testing by hand

=== "Swagger"

    1. Start the backend and open `http://127.0.0.1:8000/docs`.
    2. Log in with `POST /auth/login`, then `POST /auth/verify-otp` with the emailed code.
    3. Click **Authorize** and paste the `access_token`.
    4. Try any endpoint with **Try it out**.

=== "Postman"

    1. Import the **Umucyo** collection and the **Umucyo Local** environment.
    2. Put your password in the environment.
    3. Run the collection. The first run emails a code; put it in `otp_code` and run again.

### Testing the agent app on a phone

The agent app is used on a phone, so test it on a phone at least once before a release.

```bash
npm run dev -- --hostname 0.0.0.0
```

Open the printed network address from your phone — both devices on the same Wi-Fi. Check the things a desk cannot show you: whether the buttons are reachable with a thumb, whether the form survives the keyboard opening, and whether the page is readable outdoors at full brightness.

## What every feature should test

- [x] The normal case works.
- [x] Missing or wrong input is refused (`400` or `422`).
- [x] A user who is not signed in is refused (`401`).
- [x] A user with the wrong role is refused (`403`).
- [x] Another cooperative's data is hidden (`404`).
- [x] Doing something twice is refused or harmless (`409`).

Two additions worth having in mind while reading that list:

- **Boundaries, not just "wrong".** Zero, negative, and exactly-at-the-limit are different from garbage input, and each fails differently.
- **The database after the refusal.** A refused request that still wrote half a row is worse than one that wrote nothing. Assert the count did not change.

## Adding a case to the register

When you write a new test, add its row to the spreadsheet in the same pull request.

1. Open the [:material-table-large: test case register](https://docs.google.com/spreadsheets/d/1ohqQuJDhxkFJYOhb0hr6PcC6P6mhbrpQqFtdShplZ2U/edit?usp=sharing).
2. Take the next free ID — never reuse a number, even for a deleted case.
3. Fill every column, including **Automated?** with the test name, or `No` and the reason.
4. Set **Status** to what actually happened.
5. Do not delete a failing case to make the sheet look green. Fix it, or set it to blocked and say why.

## When a test fails

| What you see | What it usually means |
| --- | --- |
| `401` everywhere in a manual run | The token expired (30 minutes by default) — sign in again |
| `alembic heads` prints two lines | Two migrations share a parent; merge them before continuing |
| A test passes alone, fails in the full run | Shared state: a test is not cleaning up, or relying on order |
| `sqlite3.IntegrityError` in a test | The fixture is bypassing a service and writing rows directly |
| The frontend builds locally, fails in CI | An environment variable is missing — the app expects it at build time |
| A screen works locally, is stale in production | The build is cached, or the deploy did not include the change |

## Before a pull request

- [ ] All checks above pass.
- [ ] Screenshots or a recording of changed screens are in the pull request.
- [ ] `develop` is merged into your branch and conflicts are resolved.
- [ ] New behaviour has a test, or the pull request says why it does not.
- [ ] Any new test case is added to the register with its ID.

## Where to go next

- [:material-table-large: Umucyo — Test Cases](https://docs.google.com/spreadsheets/d/1ohqQuJDhxkFJYOhb0hr6PcC6P6mhbrpQqFtdShplZ2U/edit?usp=sharing) — the full register
- [Backend](backend.md) — where the rules being tested live
- [API Reference](api.md) — the endpoints and their status codes
- [Security](security.md) — why the permission cases exist
- [Deployment](deployment.md) — what to check after a release
