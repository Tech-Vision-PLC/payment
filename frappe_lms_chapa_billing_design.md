# Frappe LMS Billing and Chapa Split-Payment Integration

## 1. Project overview

We are extending Frappe LMS and its existing Payments module to support Chapa split payments. When a student purchases a course, the payment should be split between:

- **Teacher:** receives the teacher's share of the course sale.
- **Platform:** receives a percentage commission.

The billing module must record the financial breakdown and track payment and distribution statuses separately.

### Core principles

1. Frappe is the source of truth for internal billing records.
2. Chapa is the source of truth for payment and distribution outcomes reported by its API.
3. A successful student payment does not, by itself, prove that the teacher's funds have been settled.
4. Financial records must be immutable or auditable after creation.
5. Duplicate callbacks or webhooks must not create duplicate billing records or duplicate payouts.

> **Important:** Confirm the current Chapa API requirements, split-payment behaviour, fees, and account eligibility against the documentation and your enabled Chapa account before deploying live transactions.

## 2. Proposed architecture

```text
Student purchases a course
          |
          v
Frappe LMS
          |
          v
Existing Payments module
          |
          v
Chapa checkout / transaction initialization
          |
          v
Server-side payment verification
          |
          v
Billing and commission service
          |
          +-----------------------+
          |                       |
          v                       v
Teacher's share             Platform commission
          |
          v
Distribution and settlement reconciliation
```

Keep the existing Payments integration where possible. Extend it with Chapa-specific split-payment handling and a billing layer rather than replacing the payment system unnecessarily.

## 3. Payment and billing lifecycle

1. **Teacher onboarding**
   - Verify the teacher's payout information.
   - Create or register the teacher's Chapa subaccount if supported and required.
   - Store the returned Chapa subaccount ID in protected teacher payout settings.
   - Do not allow teachers to change verified payout details without an appropriate verification process.

2. **Student checkout**
   - Retrieve the course price, instructor, currency, and applicable commission rate on the server.
   - Create a unique payment reference.
   - Create a pending payment and billing record.
   - Do not trust the course price or commission sent by the browser.

3. **Initialize Chapa**
   - Send the amount, currency, unique transaction reference, required customer fields, callback/return details, and the relevant subaccount/split configuration.
   - Return the checkout URL to the frontend.
   - Keep secret keys on the server.

4. **Verify payment**
   - Verify the transaction server-side using Chapa's documented verification endpoint.
   - Do not treat the browser redirect alone as proof of payment.
   - Check the transaction reference, amount, currency, and status.

5. **Finalize billing**
   - Once payment is verified, record the gross amount, commission rate, commission amount, and teacher's share.
   - Use idempotent processing so repeated callbacks cannot duplicate financial records.
   - Keep payment status separate from distribution status.

6. **Reconcile distribution**
   - Record Chapa's reported split/distribution outcome.
   - Compare expected amounts with confirmed settlement data when available.
   - Flag mismatches for administrator review.
   - Do not mark the teacher as paid merely because the student's payment succeeded.

7. **Refunds and reversals**
   - Record refunds, chargebacks, and reversals as explicit events linked to the original transaction.
   - Reconcile any amount returned or reversed.
   - Never silently overwrite the original financial record.

## 4. Commission calculation

Example:

| Item | Amount |
|---|---:|
| Course price | ETB 1,000.00 |
| Platform commission rate | 20% |
| Platform commission | ETB 200.00 |
| Teacher's share | ETB 800.00 |
| Total before payment fees | ETB 1,000.00 |

The example excludes Chapa transaction fees. Confirm whether fees are charged to the platform, customer, teacher, or another party under the actual account configuration.

### Calculation rules

- Store money as currency values with appropriate decimal precision.
- Use Python's `Decimal` for financial calculations, not binary floating-point arithmetic.
- Capture the commission rate at purchase time so later rate changes do not alter historical transactions.
- Define and document rounding rules.
- Distinguish gross sale, platform commission, gateway fees, teacher's expected share, and actual settled amount.

### Example Python calculation

```python
from decimal import Decimal, ROUND_HALF_UP


def calculate_course_billing(course_price, commission_rate):
    price = Decimal(str(course_price))
    rate = Decimal(str(commission_rate))

    if price <= 0:
        raise ValueError("Course price must be positive")

    if not Decimal("0") <= rate <= Decimal("100"):
        raise ValueError("Commission must be between 0 and 100")

    commission = (
        price * rate / Decimal("100")
    ).quantize(
        Decimal("0.01"),
        rounding=ROUND_HALF_UP,
    )

    teacher_share = price - commission

    return {
        "gross_amount": price,
        "commission_rate": rate,
        "commission_amount": commission,
        "teacher_amount": teacher_share,
    }
```

This is an internal billing calculation example. Chapa's actual settlement may differ due to fees and split rules.

## 5. Illustrative Chapa split-payment payload

The following is a conceptual example only. It is not a complete production request. Confirm the current Chapa API schema and required fields before using it.

```json
{
  "amount": "1000",
  "currency": "ETB",
  "tx_ref": "COURSE-PAYMENT-UNIQUE-REFERENCE",
  "subaccounts": {
    "id": "TEACHER_CHAPA_SUBACCOUNT_ID",
    "split_type": "percentage",
    "split_value": 0.2
  }
}
```

In the documented split-payment examples, `split_value` represents the platform commission. Verify the exact value format and semantics against the current Chapa documentation and your account. Include required customer fields, callback URL, and return URL as specified by the API.

Never hard-code a real subaccount ID or API secret into source code. Generate a unique `tx_ref` for each payment attempt and persist it in Frappe.

## 6. Proposed Frappe DocTypes

These are proposed custom DocTypes. Exact names, links, and fields should be adjusted to match the installed Frappe, LMS, and Payments versions.

### A. Teacher Payout Settings

Purpose: store the teacher's Chapa subaccount reference and payout configuration.

Suggested fields:

- `teacher`: Link to the relevant teacher/user record
- `chapa_subaccount_id`: Data
- `payout_account_status`: Select (Pending Verification, Verified, Disabled)
- `commission_rate`: Percent or Decimal, if commission is configurable per teacher
- `verified_on`: Datetime
- `verified_by`: Link to User

Protect this DocType with role-based permissions. Avoid storing unnecessary sensitive payout data.

### B. Course Billing

Purpose: record the financial breakdown for each course payment.

Suggested fields:

| Field | Suggested type | Purpose |
|---|---|---|
| `payment_reference` | Link or Data | Link to existing payment record |
| `course` | Link | Purchased course |
| `teacher` | Link | Instructor |
| `student` | Link | Student/payer |
| `gross_amount` | Currency | Original course price |
| `currency` | Link or Select | Transaction currency |
| `commission_rate` | Percent or Decimal | Rate captured at purchase |
| `commission_amount` | Currency | Platform commission |
| `teacher_amount` | Currency | Teacher's calculated share |
| `chapa_tx_ref` | Data | Unique payment reference |
| `chapa_transaction_id` | Data | Chapa transaction ID |
| `payment_status` | Select | Pending, Paid, Failed, Refunded, Partially Refunded |
| `distribution_status` | Select | Pending, Confirmed, Failed, Review Required |
| `verified_at` | Datetime | Server-side verification time |

Use unique constraints or equivalent safeguards for payment references and event processing identifiers where appropriate.

### C. Distribution Record

Purpose: track the expected and reported distribution outcome.

Suggested fields:

- `billing_record`: Link to Course Billing
- `teacher`: Link to teacher
- `chapa_tx_ref`: Data
- `chapa_subaccount_id`: Data
- `expected_teacher_amount`: Currency
- `reported_teacher_amount`: Currency, when available from reliable gateway data
- `distribution_status`: Select
- `last_checked_at`: Datetime
- `reconciliation_notes`: Small Text

Do not invent a separate transfer operation if Chapa's selected split-payment method settles funds as part of the original transaction. Model what the actual API supports.

### D. Billing Event Log

Purpose: maintain an audit trail for external events and changes.

Suggested fields:

- `event_id`: Data, unique where available
- `event_type`: Select or Data
- `payment_reference`: Link or Data
- `chapa_tx_ref`: Data
- `received_at`: Datetime
- `processing_status`: Select
- `payload_hash`: Data
- `processing_result`: Small Text

Store only the minimum event data needed. Protect logs from unauthorized access and avoid exposing secrets or unnecessary customer details.

## 7. Dashboard requirements

### Administrator Billing Dashboard

Show:

- Gross course sales for a selected period
- Platform commission earned
- Expected teacher earnings
- Confirmed distribution/settlement totals where available
- Pending and failed payments
- Distributions needing reconciliation
- Refunds and chargebacks
- Filters for date range, course, teacher, payment status, and distribution status
- Exportable reports with appropriate permissions

### Teacher Earnings Dashboard

Show only the current teacher's own data:

- Gross sales from their courses
- Commission deducted
- Expected earnings
- Confirmed paid/settled amount where supported by gateway data
- Pending or failed distribution status
- Transaction history and refunds

Do not expose other teachers' billing records.

## 8. Security and reliability requirements

- Verify every payment server-side.
- Validate transaction reference, amount, currency, and status.
- Make webhook/callback processing idempotent.
- Use database transactions for related internal updates.
- Protect API secrets in server-side configuration.
- Use role-based permissions and server-side authorization checks.
- Keep an audit trail of important financial events.
- Never accept a teacher's subaccount ID or commission rate directly from an untrusted browser request.
- Handle retries safely.
- Reconcile expected billing amounts against gateway-reported outcomes.
- Test failed payments, duplicate callbacks, delayed callbacks, refunds, and mismatched amounts.
- Do not log API secrets or unnecessary personal/payment data.

## 9. Implementation plan

1. Inspect the existing Frappe LMS and Payments DocTypes and the current Chapa integration.
2. Identify where payment initialization and verification currently happen.
3. Confirm Chapa's current split-payment API schema, fees, account requirements, and settlement behaviour.
4. Add teacher payout settings and commission configuration.
5. Add the billing record and commission calculation service.
6. Extend Chapa initialization with the correct subaccount/split fields.
7. Implement server-side verification and idempotent event handling.
8. Implement distribution tracking and reconciliation.
9. Build administrator and teacher dashboards with strict permissions.
10. Test in the Chapa test environment before enabling live payments.

## 10. Documentation references

- Chapa Split Payments: https://docs.chapa.global/docs/v1/integrations/split-payment
- Chapa API documentation: https://docs.chapa.co/
- Frappe Learning repository: https://github.com/frappe/lms
- Frappe Payments repository: https://github.com/frappe/payments

Check the current documentation for API changes and verify that the desired split-payment feature is enabled for your Chapa account.

## 11. Next step

Review the existing Chapa transaction initialization and verification code in your Frappe Payments integration. Use that implementation to determine the exact extension points and avoid duplicating payment logic.

Before sharing code, remove API keys, tokens, customer details, and real bank information.
