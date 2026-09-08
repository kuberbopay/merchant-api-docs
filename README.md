# KuberoPay Merchant API

A merchant-facing integration guide for accepting payments with KuberoPay.
This folder is the source for the merchant developer documentation.
It explains how a merchant integrates KuberoPay into their own platform: create an API key, generate a payment link (order), check its status, list payments, refund, and receive webhooks.

There are no secrets in this folder.
Every key shown is an obvious placeholder such as `sk_test_9Fh2xxxx4Q2f`.

## What is here

- `index.html` - the full guide as a single self-contained page, ready to serve or host.
- `README.md` - this guide in Markdown, the maintainable source.
- `assets/` - reserved for images used by the docs.

The hosted version of `index.html` is published at:
https://claude.ai/code/artifact/03f5b981-ecdd-40fa-8efb-9527a1682b3f

## Base URL

All API calls go to the partner API over HTTPS.
KuberoPay runs two separate environments, each on its own host.

```
Production:  https://api.kuberopay.com/api/v1      (sk_live_ keys, real money)
UAT/sandbox: https://uatapi.kuberopay.com/api/v1   (sk_test_ keys, no real money)
```

Build against UAT, then switch the base URL and key to Production to go live.
Each environment issues its own keys, so a test key never reaches production.

## 1. Create your API key

API keys are issued from the KuberoPay merchant console, not from the API itself.
You need a team member with the Merchant admin role.

Keys are per environment, so you do this once for UAT and again for Live.
The exact page is **Developers -> API keys**:

```
UAT/sandbox: https://uatapp.kuberopay.com/api/keys
Live:        https://app.kuberopay.com/api/keys
```

1. Sign in to the console for the environment you want and open **Developers -> API keys** (the URL above).
2. Select **Issue key**.
   The mode (Test or Live) is fixed by which console you are in, so a UAT key can never move real money.
3. Copy the secret key from the dialog and store it in your server-side secret manager.
   It is shown only once and cannot be retrieved later.
4. **Yes, you create a key per environment.**
   Make a test key in UAT while you build, then a separate live key in the live console before going live.
   Keys never carry between environments.
5. If a key is ever exposed, revoke it from the same screen and issue a new one.

Keep secret keys server-side.
A key starting `sk_` can move money.
Never ship it in browser, mobile, or any client-side code, and never commit it to a repository.

(The `index.html` guide includes an annotated picture of this exact screen.)

## 2. Authentication

Every request is authenticated with your secret key as a Bearer token.

```
Authorization: Bearer sk_test_9Fh2xxxxxxxxxxxxxxxx4Q2f
```

**Scopes.**
A newly issued key carries `payment:read`, `payment:write`, and `refund:write`.
Payout scopes are granted separately.

**Idempotency.**
Send an `Idempotency-Key` header on every money-moving `POST` (payment intents, refunds, payouts).
Retrying the same key returns the original result instead of creating a duplicate.

**Request signing.**
Money mutations - confirming an intent, creating a refund or a payout - must also carry an `X-KP-Signature` header.
Read requests need only the Bearer key.

```
signature = HMAC_SHA256(key = "<your secret key>", message = "{t}.{rawBody}")
X-KP-Signature: t=1737331200,v1=<hex>
```

`t` is the current unix time in seconds and the signature is valid for five minutes.
This inbound scheme is keyed with your API key.
It is different from the outbound webhook signature below, which is keyed with a per-endpoint secret.
Do not mix them up.

## 3. Create a payment link

A payment link is the simplest way to take a payment and is your order-creation call.
Post the amount and the customer, and receive a hosted checkout URL to hand to the buyer.

```
POST /payment_links      (scope: payment:write)
```

Request body:

| Field | Type | Notes |
|---|---|---|
| `amount` (required) | string | Decimal amount, e.g. `"499.00"`. |
| `currency` (required) | string | ISO code, e.g. `"INR"`, `"USD"`, `"EUR"`. |
| `customer` (required) | object | `{ name, email, phone }`, all three required. |
| `description` | string | Shown to the buyer on checkout. |
| `reference_id` | string | Your own order id, echoed back and searchable. |
| `expires_at` | string | ISO time after which the link cannot be paid. |
| `allowed_methods` | string[] | Restrict methods, e.g. `["card","upi","crypto"]`. Omit to allow all enabled methods. |
| `success` | object | `{ redirect_url, message }`. |
| `metadata` | object | Any key/value pairs returned with the link. |

Example:

```bash
curl https://api.kuberopay.com/api/v1/payment_links \
  -H "Authorization: Bearer sk_test_9Fh2xxxxxxxx4Q2f" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "499.00",
    "currency": "INR",
    "customer": { "name": "Asha Rao", "email": "asha@example.com", "phone": "+919800000000" },
    "description": "Order #A-1042",
    "reference_id": "A-1042",
    "allowed_methods": ["card", "upi", "crypto"]
  }'
```

Response (`201 Created`):

```json
{
  "id": "plink_01JBQ8N3K2Y7",
  "object": "payment_link",
  "short_code": "kp_7Yq3Za",
  "url": "https://checkout.kuberopay.com/?code=kp_7Yq3Za",
  "amount": "499.00",
  "currency": "INR",
  "status": "active",
  "reference_id": "A-1042",
  "allowed_methods": ["card", "upi", "crypto"],
  "paid_at": null,
  "created_at": "2026-01-20T10:00:00Z"
}
```

Send the buyer to `url`.
KuberoPay hosts the checkout, presents the enabled methods, and collects the payment.

## 4. Check payment status

Retrieve a link to see whether it has been paid.
The response embeds the linked payment and its lifecycle status.

```
GET /payment_links/{id}      (scope: payment:read)
```

```json
{
  "id": "plink_01JBQ8N3K2Y7",
  "status": "paid",
  "paid": true,
  "reference_id": "A-1042",
  "payment_intent": { "id": "pay_01JBQ8P0RA5T", "status": "succeeded", "amount": "499.00" },
  "latest_payment_intent": { "id": "pay_01JBQ8P0RA5T", "status": "succeeded", "payment_method": "crypto" }
}
```

Polling is fine for low volume.
Webhooks are the recommended way to learn the moment a payment succeeds, especially for crypto where confirmation takes minutes.

## 5. List payments

```
GET /payment_links      (scope: payment:read)
```

Query parameters: `page`, `pageSize`, `status` (comma-separated), `currency`, `created_from`, `created_to`, `q`.
The response returns `{ data[], page, pageSize, total }`.

```bash
curl "https://api.kuberopay.com/api/v1/payment_links?status=paid&pageSize=20" \
  -H "Authorization: Bearer sk_test_9Fh2xxxxxxxx4Q2f"
```

## 6. Refund a payment

Refund a captured payment in full or in part.
A refund enters review and settles once approved, so its status starts at `pending`.

```
POST /refunds      (scope: refund:write, signed, idempotent)
```

```bash
curl https://api.kuberopay.com/api/v1/refunds \
  -H "Authorization: Bearer sk_test_9Fh2xxxxxxxx4Q2f" \
  -H "Idempotency-Key: A-1042-refund-1" \
  -H "X-KP-Signature: t=1737331200,v1=<hex>" \
  -H "Content-Type: application/json" \
  -d '{ "payment_intent": "pay_01JBQ8P0RA5T", "amount": "499.00", "reason": "customer_request" }'
```

## 7. Webhooks

A webhook is how KuberoPay tells your server the moment a payment changes.
It is the reliable way to learn an outcome without polling.
You register one endpoint URL per environment in the console, and KuberoPay then POSTs a signed event to it on every payment change.

### Set it up (once per environment)

The webhooks page is **Developers -> Webhooks**:

```
UAT/sandbox: https://uatapp.kuberopay.com/api/webhooks
Live:        https://app.kuberopay.com/api/webhooks
```

1. Open **Developers -> Webhooks** in the console for the environment you want (the URL above).
2. Select **Add endpoint** and enter your own HTTPS URL that will receive events, e.g. `https://yourshop.com/webhooks/kuberopay`.
3. Choose which events to receive, or keep the default set (it already includes the payment events).
   Save.
4. Copy the **signing secret** shown for the endpoint.
   You use it to verify every delivery (below).
   It is shown once; you can rotate it later from the same screen.
5. Use **Send test event** to fire a sample delivery and confirm your endpoint replies `2xx`.

Event envelope:

```json
{
  "id": "evt_01JBQ9V7C1",
  "type": "payment.succeeded",
  "created_at": "2026-01-20T10:03:11Z",
  "data": { "payment_id": "pay_01JBQ8P0RA5T", "amount": "499.00", "currency": "INR" }
}
```

**Verify the signature.**
Each delivery carries `x-kp-signature` (a hex HMAC of the raw body, keyed with your endpoint's signing secret) and `x-kp-event-id` for de-duplication.
Compute the same HMAC over the raw request body and compare in constant time before trusting the event.
Delivery is at-least-once, so treat repeated `x-kp-event-id` values as one.

```js
const crypto = require("crypto");

function verify(rawBody, headerSig, signingSecret) {
  const expected = crypto.createHmac("sha256", signingSecret).update(rawBody).digest("hex");
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(headerSig));
}
```

Events you can subscribe to:

- Payments: `payment.succeeded`, `payment.processing`, `payment.requires_action`, `payment.requires_capture`, `payment.captured`, `payment.failed`, `payment.canceled`.
- Crypto pay-in: `crypto.payment.detected`, `crypto.payment.confirming`, `crypto.payment.completed`, `crypto.payment.underpaid`, `crypto.payment.expired`.
- Refunds: `refund.created`, `refund.succeeded`, `refund.failed`, `refund.canceled`.
- Settlement: `settlement.settled`, `settlement.failed`, `crypto_conversion.credited`, `rolling_reserve.released`.
- Disputes: `dispute.opened`, `dispute.under_review`, `dispute.won`, `dispute.lost`.
- Account: `merchant.approved`, `merchant.kyc.approved`, `merchant.kyc.rejected`.

## 8. Status reference

Payment (intent) status: `created`, `requires_action`, `processing`, `requires_capture`, `partially_captured`, `succeeded`, `failed`, `canceled`, `expired`.

Refund status: `pending`, `processing`, `succeeded`, `failed`, `canceled`.

Crypto deposit status: `awaiting`, `detected`, `confirming`, `completed`, `underpaid`, `expired`.

## 9. Errors

Errors return a standard problem document with an HTTP status and a machine-readable code.
Branch on the `code`, not the message text.

| HTTP | When | What to do |
|---|---|---|
| 401 | Missing, wrong, or wrong-mode key. | Check the Bearer key and that its mode matches your environment. |
| 403 | `merchant_not_approved`, or a scope the key lacks. | Complete onboarding, or issue a key with the needed scope. |
| 409 | An `Idempotency-Key` was reused with a different body. | Use a fresh key for a new request. |
| 422 | Validation failed on a field. | Fix the named field and retry. |
| 429 | Rate limited. | Back off and retry after the indicated window. |

## Serving the HTML

`index.html` is self-contained (only Google Fonts is loaded externally).
Serve it as a static file from any web server or CDN, or open it directly in a browser.
It follows the system theme and includes a light/dark toggle.
