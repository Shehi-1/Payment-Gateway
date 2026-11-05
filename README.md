# Payments Lab

Payments Lab is a full-stack learning sandbox inspired by [“Design a Payment System - System Design Interview”](https://www.youtube.com/watch?v=olfaBgJrUBI). It has a browser dashboard, a Node.js HTTP API, and a local SQLite database. It **never contacts a real payment gateway, collects card numbers, or moves real money**.

## Run locally

Requires Node.js 24 or newer. No packages need to be installed.

```powershell
node server.js
```

Open [http://localhost:3000](http://localhost:3000). The API listens on port 3000 by default; set `PORT` to use another port. Sandbox state persists in `data/payments.db` across restarts. Use **Reset sandbox** in the dashboard to restore the starting balances and clear demo payments, events, and ledger activity.

Run the test suite with:

```powershell
node --test
```

## Try the gateway scenarios

Create a payment from one sample account to another and select a mock gateway outcome:

- **Approved test card:** the mock gateway returns approval code `00`; the payment settles and posts matching debit and credit entries.
- **Issuer decline test card:** the mock gateway returns decline code `05`; the payment fails and its reserved funds are released.
- **Insufficient funds test card:** the mock gateway returns decline code `51`.

These are scenario names, not card numbers or credentials. Do not enter real card information. The API accepts a `testCardId` only; no PAN or CVC field exists. Payments are simulated account transfers that let you follow the server-side authorization and settlement flow.

## API

- `GET /api/health` — health and sandbox processor mode
- `GET /api/state` — accounts, payments, test-card scenarios, ledger, events, and metrics
- `GET /api/payments/:id` — retrieve one payment
- `POST /api/payments` — validate and create an idempotent payment
- `POST /api/reset` — restore sample balances and clear sandbox activity

Example request:

```json
{
  "fromAccountId": "alice",
  "toAccountId": "northstar",
  "amount": "24.00",
  "idempotencyKey": "order-123-attempt-1",
  "testCardId": "sandbox_approved"
}
```

Example using PowerShell:

```powershell
$body = @{
  fromAccountId = "alice"
  toAccountId = "northstar"
  amount = "24.00"
  idempotencyKey = "order-123-attempt-1"
  testCardId = "sandbox_approved"
} | ConvertTo-Json

Invoke-RestMethod -Uri http://localhost:3000/api/payments `
  -Method Post -ContentType "application/json" -Body $body
```

Reuse the same idempotency key and request to get the original payment without creating another one. Reusing the key with different payment details returns `409 Conflict`. Newly accepted payments return `processing`, then asynchronously become `succeeded` or `failed`.

## What this demonstrates

1. **API validation and idempotency:** the backend validates amounts and accounts, checks available balance, and prevents duplicate requests.
2. **Payment authorization:** the sandbox gateway returns a deterministic test result and response code asynchronously.
3. **Webhook-style processing:** a local mock callback records the gateway result and updates the payment lifecycle.
4. **Atomic settlement:** SQLite commits the payment status, audit events, and equal debit/credit ledger entries together.
5. **Persistence:** payments, event history, and ledger entries survive a server restart. Any payment left processing at shutdown is scheduled for mock processing after restart.

The project is intentionally educational, **not production payment software**. The callback is simulated inside the server, not a real signed webhook. There is no authentication, real card handling, network gateway, durable job queue, chargeback handling, or compliance implementation.
