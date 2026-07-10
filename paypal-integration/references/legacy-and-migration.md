# Legacy and migration guidance

Use this reference when the task involves older PayPal integrations, compatibility review, or migration planning. First inventory the existing SDKs, endpoints, notification mechanisms, stored PayPal IDs, historical transaction requirements, and cutover constraints before recommending removal or replacement.

## Legacy or maintenance-focused patterns

Treat the following as legacy or migration-focused unless the user explicitly targets an existing implementation:

- REST Payments v1 checkout flows using `/v1/payments/payment`.
- Code built around `Payment.find(...)`, `payment.execute(...)`, payer ID execution, sale IDs, or SDK wrappers that model REST Payments v1.
- Classic Express Checkout NVP/SOAP APIs, including `SetExpressCheckout`, `GetExpressCheckoutDetails`, and `DoExpressCheckoutPayment`.
- IPN-first fulfillment.
- PDT-only success-page validation.
- Older Billing Plans / Billing Agreements under Payments v1.
- Older Smart Buttons examples when the user asks specifically for JavaScript SDK v6.

Do not call a PayPal product deprecated unless PayPal documentation or supplied context explicitly says so. Use **legacy** or **maintenance-only** when the status is operationally old but not confirmed as deprecated.

## REST Payments v1 warning

Do not generate this pattern for new checkout work:

```python
from paypalrestsdk import Payment

payment = Payment.find(payment_id)
payment.execute({"payer_id": payer_id})
```

Problems:

- It targets REST Payments v1 concepts rather than Orders API v2.
- It encourages payer-ID execution instead of server-owned order capture/authorization.
- It can confuse payment ID, order ID, payer ID, authorization ID, sale ID, and capture ID.
- It does not match JavaScript SDK v6 checkout-session patterns.

Use this shape for new one-time checkout work:

```http
POST /v2/checkout/orders
POST /v2/checkout/orders/{order_id}/capture
GET  /v2/payments/captures/{capture_id}
POST /v2/payments/captures/{capture_id}/refund
```

## Older Smart Buttons pattern

Older JavaScript integrations commonly use:

```javascript
paypal.Buttons({
  createOrder(data, actions) {
    return actions.order.create({ /* order payload */ });
  },
  onApprove(data, actions) {
    return actions.order.capture();
  }
}).render("#paypal-button-container");
```

This is not the JavaScript SDK v6 default. For SDK v6, use the v6 core script, `window.paypal.createInstance(...)`, custom elements, eligibility checks, payment sessions, `session.start(...)`, and a server-created order promise resolving to `{ orderId: "..." }`.

## IPN and PDT

Use IPN or PDT only for existing integrations that depend on them.

If maintaining IPN:

1. POST the received form payload back to the matching PayPal IPN verification endpoint with `cmd=_notify-validate`.
2. Process only `VERIFIED` responses.
3. Validate receiver email/merchant ID, amount, currency, item/reference data, transaction ID, and environment.
4. Deduplicate by `txn_id`.
5. Map IPN status to internal state conservatively.
6. Plan migration to webhooks for new flows.

If maintaining PDT:

- Treat it as return-page validation, not a complete asynchronous reconciliation mechanism.
- Do not fulfill solely from PDT when webhook/IPN-style asynchronous state is required.

## Migration path

1. Inventory current flows, SDKs, endpoints, IDs, and notification mechanisms.
2. Classify each flow as current, legacy-maintenance, deprecated, or unknown.
3. Add server endpoints for Orders API v2 create/capture or authorize/capture-later.
4. Add webhook verification, deduplication, logging, and resource-state reconciliation.
5. Add JavaScript SDK v6 browser flow with payment sessions and server-created orders.
6. Route new checkout traffic to the new flow while preserving legacy handlers for existing transactions.
7. Reconcile old IDs and new IDs in reporting and support tooling.
8. Remove legacy code only after refund, dispute, chargeback, tax, and accounting retention requirements are satisfied.

## Migration audit checklist

- Identify whether the code uses SDK v5, SDK v6, REST Payments v1, Orders v2, Payments v2, NVP/SOAP, IPN, PDT, or webhooks.
- List all PayPal IDs persisted by the system and their meaning.
- Check whether the browser creates or captures orders.
- Check whether the server validates amount, currency, ownership, and PayPal status before fulfillment.
- Check whether webhook/IPN/PDT handlers verify source and deduplicate events.
- Check whether refunds use capture IDs or legacy sale IDs.
- Check whether subscriptions use current Billing Subscriptions APIs or older billing agreements.
- Check whether account/product eligibility is assumed without evidence.

## Status labels

Use these labels consistently:

- **Current/supported:** Suitable for new work when account, region, currency, and product eligibility are satisfied.
- **Legacy-maintenance:** May still appear in production or older documentation, but should not be the default for new integrations.
- **Deprecated:** Explicitly marked by PayPal or supplied authoritative documentation as deprecated or replaced.
- **Unknown/verify:** Status cannot be established from available context.
