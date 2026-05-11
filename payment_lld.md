# Payment System LLD — Final Mental Model

Senior engineers do NOT start with SQL.

They derive schema from business reality.

---

# 1. What is the core thing?

Ask:

> What is fundamentally happening?

Core abstraction:

# Money movement attempt

So:

```text id="f4x2mp"
Payment
```

NOT:

```text id="d8q1rv"
StripePayment
UPIPayment
CardPayment
```

Those are implementation details.

---

# Deeper abstraction

Money moves:

```text id="j6m9wa"
FROM somewhere
TO somewhere
```

So we need:

# Account

Because money belongs to accounts.

Not directly users.

This survives:

* wallets
* merchant balances
* escrow
* cashback
* platform fees
* taxes
* split payouts

---

# 2. What exists independently?

These become entities.

---

# User

```text id="w1r8pk"
User
- id
- name
```

---

# Merchant

```text id="q5x2mv"
Merchant
- id
- name
```

---

# Account

```text id="z9m4wa"
Account
- id
- ownerType
- ownerId
- currency
```

Represents money ownership.

---

# Order

```text id="t3q7xp"
Order
- id
- totalAmount
- status
```

Exists before payment.

---

# PaymentMethod

```text id="v8m1qk"
PaymentMethod
- id
- type
- provider
```

Avoids branching chaos.

---

# Payment

Core workflow entity.

```text id="u4x6pr"
Payment
- id
- orderId
- payerAccountId
- payeeAccountId
- amount
- status
- provider
```

IMPORTANT:

Payment is NOT a junction schema.

Because it represents:

* workflow
* retries
* provider interaction
* lifecycle
* authorization
* failures

---

# Refund

Independent lifecycle.

```text id="c7m2qx"
Refund
- id
- paymentId
- amount
- status
```

---

# Settlement

```text id="n1x9wp"
Settlement
- id
- paymentId
- amount
- status
```

---

# LedgerEntry

Most important financial entity.

```text id="m5q8rv"
LedgerEntry
- id
- accountId
- entryType
- amount
```

Tracks actual balance movement.

---

# 3. What connects things?

---

# User ↔ Account

```text id="r2m7wa"
User 1 --- N Account
```

---

# Merchant ↔ Account

```text id="x6q1pk"
Merchant 1 --- N Account
```

---

# Order ↔ Payment

```text id="f9m4vx"
Order 1 --- N Payment
```

Because retries happen.

---

# User ↔ PaymentMethod

```text id="p3x8qr"
User 1 --- N PaymentMethod
```

---

# Payment ↔ PaymentMethod

```text id="a7m2wk"
Payment N --- 1 PaymentMethod
```

---

# Payment ↔ Account

Money moves:

```text id="u8q5nz"
FROM account
TO account
```

So:

```text id="v1m9qp"
Payment
- payerAccountId
- payeeAccountId
```

Again:

NOT junction schema.

Because Payment is workflow/business entity.

---

# 4. What is ONLY relationship mapping?

Now derive actual junction schemas.

---

# Split payouts

Example:

₹1000 payment split into:

| Recipient | Amount |
| --------- | ------ |
| Merchant  | 850    |
| Platform  | 100    |
| Tax       | 50     |

Now relationship itself carries data.

So:

```text id="q4x1mr"
PaymentRecipient
- paymentId
- accountId
- amount
```

This IS junction schema.

Because it answers:

```text id="w7m3pk"
which account receives how much
```

---

# Key junction rule

Ask:

> Does attribute depend on BOTH entities together?

Example:

```text id="j5q9vx"
payment + account -> amount
```

YES.

So amount belongs in relationship schema.

---

# Compare with WhatsApp

---

# ConversationParticipant

```text id="n8m2qa"
ConversationParticipant
- conversationId
- userId
```

Only answers:

```text id="r1x6wp"
who belongs where
```

No message field.

Because Message is independent workflow/event.

---

# Message

```text id="m4q7pk"
Message
- conversationId
- senderId
- content
```

Represents:

```text id="x2m9vr"
something happened
```

---

# SAME distinction in payments

---

# PaymentRecipient.amount

Relationship metadata.

---

# Payment

Workflow/event entity.

---

# 5. What happens over time?

Payments evolve:

```text id="b6x3qp"
INITIATED
PENDING
AUTHORIZED
SUCCESS
FAILED
REFUNDED
```

So we need:

```text id="k9m5wa"
PaymentEvent
- paymentId
- eventType
- payload
- createdAt
```

This gives:

* auditability
* retries
* debugging
* reconciliation
* history

---

# Refund lifecycle

Possible:

```text id="t1q8mv"
RefundEvent
```

---

# Settlement lifecycle

Possible:

```text id="v5x2pr"
SettlementEvent
```

---

# 6. Where does state belong?

Avoid giant chaos tables.

---

# WRONG

```text id="u7m1qx"
Payment
- refundStatus
- settlementStatus
- webhookStatus
- reconciliationStatus
```

---

# CORRECT

Each lifecycle owns its own state.

---

# Payment owns payment state

```text id="c3q9wp"
Payment.status
```

---

# Refund owns refund state

```text id="m8x4vr"
Refund.status
```

---

# Settlement owns settlement state

```text id="q2m7pk"
Settlement.status
```

---

# 7. Am I creating branching chaos?

Avoid:

```python id="x5q1mn"
if provider == "stripe"
```

everywhere.

Use abstraction:

```text id="v9m3qx"
PaymentProcessor
```

Implementations:

```text id="f2x8wp"
StripeProcessor
RazorpayProcessor
PaypalProcessor
```

---

# FINAL PAYMENT LLD

---

# Independent entities

```text id="r4q6mv"
User
Merchant
Account
Order
PaymentMethod
Payment
Refund
Settlement
LedgerEntry
```

---

# Junction schemas

```text id="n7m2xp"
PaymentRecipient
```

(optional for split distributions)

---

# Event entities

```text id="y1q9wr"
PaymentEvent
RefundEvent
SettlementEvent
```

(optional depending on complexity)

---

# FINAL DESIGN RULES

---

# Rule 1

Ask:

> What is independent workflow/business process?

Examples:

```text id="m5x3qp"
Payment
Message
Shipment
Refund
```

NOT junction schemas.

---

# Rule 2

Ask:

> What only exists to connect entities?

Examples:

```text id="u8q1vk"
ConversationParticipant
PaymentRecipient
Enrollment
UserRole
```

Relationship-centric schemas.

---

# Rule 3

Ask:

> Does attribute depend on BOTH entities together?

If YES → relationship/junction attribute.

Examples:

```text id="x4m7pr"
grade
splitAmount
role
joinedAt
```

---

# Rule 4

Workflow/event ≠ relationship metadata

Workflow:

```text id="a9q2wx"
Payment
Message
Refund
```

Relationship metadata:

```text id="b3m8vq"
allocation amount
membership role
grade
```

---

# Rule 5

Independent lifecycle → independent entity

Examples:

```text id="r6x1pk"
Refund
Settlement
DeliveryState
```

---

# Rule 6

Events model time evolution

Examples:

```text id="t5q9mv"
PaymentEvent
MessageEvent
```

---

# Rule 7

Design abstractions that survive future requirements

Good abstraction:

```text id="v2m4wr"
Account
```

Bad abstraction:

```text id="n8q1xp"
receiverUserId
```

---

# IMPORTANT PAYMENT SQL QUESTIONS

These are the actual SQL patterns backend/payment engineers repeatedly solve.

---

# 1. Find latest payment attempt per order

Tests:

* retries
* window functions
* latest state reasoning

```sql id="f7m3qx"
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY order_id
               ORDER BY created_at DESC
           ) rn
    FROM payments
) t
WHERE rn = 1;
```

---

# 2. Orders without successful payment

Tests:

* LEFT JOIN
* NULL reasoning

```sql id="u1x8pk"
SELECT o.id
FROM orders o
LEFT JOIN payments p
ON o.id = p.order_id
AND p.status = 'SUCCESS'
WHERE p.id IS NULL;
```

---

# 3. Find retry payments

```sql id="q5m2vr"
SELECT order_id, COUNT(*)
FROM payments
GROUP BY order_id
HAVING COUNT(*) > 1;
```

---

# 4. Total merchant revenue

```sql id="m9x4wp"
SELECT
    payee_account_id,
    SUM(amount)
FROM payments
WHERE status = 'SUCCESS'
GROUP BY payee_account_id;
```

---

# 5. Detect over-refunds

Very real fintech reconciliation problem.

```sql id="t3q7vx"
SELECT
    p.id,
    p.amount,
    SUM(r.amount)
FROM payments p
JOIN refunds r
ON p.id = r.payment_id
GROUP BY p.id, p.amount
HAVING SUM(r.amount) > p.amount;
```

---

# 6. Payments stuck in pending

```sql id="x8m1qr"
SELECT *
FROM payments
WHERE status = 'PENDING'
AND created_at < NOW() - INTERVAL '15 MINUTES';
```

---

# 7. Missing settlements

```sql id="r4q9wp"
SELECT p.id
FROM payments p
LEFT JOIN settlements s
ON p.id = s.payment_id
WHERE p.status = 'SUCCESS'
AND s.id IS NULL;
```

---

# 8. Latest payment event per payment

```sql id="v6m2pk"
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY payment_id
               ORDER BY created_at DESC
           ) rn
    FROM payment_events
) t
WHERE rn = 1;
```

---

# 9. Compute account balance from ledger

THIS is real financial-system SQL.

```sql id="k1x7mv"
SELECT
    account_id,
    SUM(
        CASE
            WHEN entry_type = 'CREDIT'
            THEN amount
            ELSE -amount
        END
    ) AS balance
FROM ledger_entries
GROUP BY account_id;
```

---

# 10. Payment success rate by provider

```sql id="z5q3wr"
SELECT
    provider,
    COUNT(*) FILTER (
        WHERE status = 'SUCCESS'
    ) * 100.0 / COUNT(*) AS success_rate
FROM payments
GROUP BY provider;
```

---

# 11. Detect duplicate payments

```sql id="b2m8qx"
SELECT
    order_id,
    amount,
    COUNT(*)
FROM payments
GROUP BY order_id, amount
HAVING COUNT(*) > 1;
```

---

# 12. Time taken from initiation to success

Tests:

* events
* analytics
* state transitions

```sql id="j7q4vp"
SELECT
    payment_id,

    MIN(created_at) FILTER (
        WHERE event_type = 'INITIATED'
    ) AS initiated_at,

    MIN(created_at) FILTER (
        WHERE event_type = 'SUCCESS'
    ) AS success_at

FROM payment_events
GROUP BY payment_id;
```

---

# Most Important Final Insight

Strong backend engineers think in:

```text id="p9m1wx"
business meaning
```

NOT:

```text id="d4q8vr"
table shapes
```

That is the real transition from CRUD developer → systems engineer.
