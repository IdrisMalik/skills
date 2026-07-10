# Implementation checklist

Use this checklist when validating a PayPal implementation, audit, migration, or debugging task.

## Discovery

- Identify the flow: checkout, authorize/capture, subscription, refund, payout, dispute, invoice, report, identity/client token, wallet/card field, platform/partner, or migration.
- Identify SDK/API surfaces: JavaScript SDK v6, older JavaScript SDK/Smart Buttons, Orders API v2, Payments API v2, supported REST API v1 product APIs, REST Payments v1, NVP/SOAP, IPN, PDT, third-party wrappers.
- Identify runtime/framework, package dependencies, PayPal script URLs, REST client code, webhook routes, database tables, environment variables, and tests.
- Identify sandbox/live separation for credentials, webhook IDs, URLs, account IDs, app IDs, and merchant IDs.
- Mark unavailable details as missing information rather than assuming them.

## Architecture

- Browser code initializes SDK components, checks eligibility, starts payment sessions, and handles approve/cancel/error callbacks.
- Server code owns cart totals, discounts, tax, shipping, currency, invoice IDs, item IDs, customer/session ownership, and fulfillment decisions.
- Server creates orders/subscriptions and captures, authorizes, refunds, cancels, revises, or pays out.
- Webhooks are verified, deduplicated, persisted or queued durably, and mapped to idempotent internal state transitions.
- Return URLs and browser callbacks are not treated as final payment confirmation.
- Legacy handlers are preserved only where needed for existing traffic, records, refunds, disputes, accounting, or migration windows.

## Security

- Client secret is never exposed to browser code.
- Server OAuth access token is never exposed to browser code.
- Browser-safe client ID is used for ordinary browser SDK initialization.
- Client token is generated server-side only for flows that require it.
- Capture/refund/payout/cancel/revise/dispute endpoints require authenticated and authorized callers.
- Capture endpoints verify that the PayPal order belongs to the authenticated checkout session or user.
- Webhook handler verifies signature, webhook ID, event environment, and duplicate event IDs.
- Privileged operations enforce role checks and audit logging.
- Raw card data is not handled unless the implementation intentionally accepts PCI scope and has a compliant design.

## Money and state validation

- Money is represented as fixed-decimal strings or decimal types, not binary floats.
- Amount and currency are recomputed or retrieved from trusted server state.
- Capture response is checked for expected amount, currency, status, PayPal order ID, and capture ID.
- Refund response is linked to the original capture ID and internal refund request.
- Subscription access is based on verified subscription state, not a return URL alone.
- Payout reconciliation checks item-level status, not only batch status.
- Internal state is separate from PayPal state and records the source of each transition.
- Duplicate browser callbacks, duplicate webhook deliveries, and out-of-order events are safe.

## Data model

Store relevant IDs separately and include a type discriminator when a shared table is unavoidable:

- Internal order/subscription/payout/refund/dispute ID
- PayPal order ID
- PayPal capture ID
- PayPal authorization ID
- PayPal refund ID
- PayPal subscription ID
- PayPal plan/product ID
- PayPal payout batch ID and payout item ID
- PayPal dispute ID
- Invoice/custom/reference ID
- Webhook event ID
- Environment: sandbox or live
- Merchant/account context for partner or marketplace flows

## Common findings

| Finding | Risk | Required change |
|---|---:|---|
| Browser creates amount or item totals | High | Create PayPal order on the server from trusted cart state. |
| Browser captures payment | High | Capture or authorize on the server and validate response. |
| Uses `paypal.Buttons(...)` while targeting SDK v6 | Medium | Replace with SDK v6 initialization, custom element, eligibility check, and session flow. |
| Uses `paypalrestsdk.Payment.find(...)` for new checkout | High | Replace REST Payments v1 flow with Orders API v2 and Payments API v2. |
| Fulfills from return URL or `onApprove` only | High | Fulfill after verified server capture, subscription state, or webhook reconciliation. |
| Webhook accepted without verification | High | Verify signature, deduplicate event ID, and reconcile resource state. |
| Refund uses PayPal order ID | Medium | Refund by capture ID. |
| Subscription grants access before state verification | High | Verify subscription through API and webhooks. |
| Payout treated as instant success | Medium | Reconcile payout item statuses. |
| Sandbox/live configuration mixed | High | Separate credentials, webhook IDs, URLs, app IDs, and account IDs. |
| PayPal ID types stored in one ambiguous column | Medium | Store IDs separately or include a strict type discriminator. |

## Validation sequence

Run or specify checks relevant to the selected flow:

1. Create order/subscription/payout/refund from server-owned state.
2. Approve checkout with sandbox buyer where applicable.
3. Capture or authorize on the server.
4. Verify amount, currency, status, ownership, and PayPal IDs.
5. Receive and verify webhook.
6. Retry browser callback.
7. Retry webhook delivery.
8. Attempt duplicate capture or duplicate fulfillment.
9. Cancel checkout.
10. Simulate capture denial or failed payment where supported.
11. Test full refund and partial refund where business rules allow.
12. Test subscription activation, renewal, failed payment, suspension, cancellation, and access revocation where applicable.
13. Test payout batch and payout item reconciliation where applicable.
14. Test dispute event/state handling where applicable.
15. Verify live-mode configuration without exposing live credentials.

## Audit output

Use this format for compatibility or production-readiness audits:

1. **Verdict:** compatible / partially compatible / incompatible / insufficient information
2. **Target integration:** product, SDK/API version, framework/runtime, account role, environment
3. **Findings:** concise technical issues with severity
4. **Required changes:** exact API, design, or code changes
5. **Legacy/deprecation notes:** legacy patterns and status assumptions
6. **Missing information:** code, product target, SDK/API version, framework, country/currency, account role, PayPal app/dashboard settings, webhook configuration, or official documentation needed for a reliable assessment
