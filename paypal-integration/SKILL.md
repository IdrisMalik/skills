---
name: paypal-integration
description: plan, implement, review, debug, or migrate paypal integrations. use for paypal javascript sdk v6, rest api v2, supported rest api v1 product apis, checkout, subscriptions, refunds, payouts, webhooks, disputes, identity or client-token flows, legacy migration, sdk/api compatibility review, idempotency, reconciliation, and production-readiness checks.
---

# PayPal Integration

Use this skill for PayPal integration tasks that require structured technical analysis or implementation. Prefer current PayPal integration patterns for new work, preserve legacy behavior when maintaining existing systems, and label API generation and status explicitly.

## Core rules

1. **Classify before changing code.** Identify product flow, API generation, SDK version, runtime, framework, account role, environment, country/currency assumptions, and whether the task is new implementation, review, debugging, or migration.
2. **Use the available context first.** Inspect user-provided code, configuration, logs, docs, routes, package files, and tests before proposing changes. Do not infer missing implementation details when they can be read from available files.
3. **Choose the PayPal surface deliberately.** Use JavaScript SDK v6 for new browser checkout UI/session flows; use Orders API v2 and Payments API v2 for one-time payment lifecycle work; use current REST API v1 product APIs where PayPal still exposes active product families.
4. **Separate trust boundaries.** Browser code may initialize SDK components, check eligibility, start sessions, and receive approval callbacks. Server code must create orders/subscriptions, capture or authorize payments, create refunds or payouts, verify webhooks, and decide fulfillment.
5. **Do not trust browser payment state.** Validate amount, currency, ownership, PayPal status, subscription state, capture IDs, and webhook data server-side before fulfillment or access changes.
6. **Make state transitions idempotent.** Use supported idempotency headers, store PayPal resource IDs by type, deduplicate webhooks, and make capture, fulfillment, refund, subscription, payout, and dispute transitions safe to retry.
7. **Preserve secrets.** Never request or expose client secrets, live credentials, OAuth access tokens, webhook signing material, or raw card data. Use placeholders and environment variables in examples.
8. **Verify uncertain details.** Product eligibility, regional support, account approval, webhook event names, SDK method signatures, and deprecation status can change. Require official PayPal verification when these affect production behavior.
9. **Label legacy patterns neutrally.** Treat REST Payments v1 checkout, `paypalrestsdk.Payment.find(...)`, `payment.execute(...)`, NVP/SOAP Express Checkout, IPN-first fulfillment, PDT-only validation, and older Smart Buttons as legacy-maintenance or migration patterns unless the task explicitly targets them.

## Integration workflow

Use this process when a task requires analysis, implementation, review, debugging, or migration. Adapt the depth to the available context and requested output.

1. **Inventory**
   - Locate PayPal-related files, routes, SDK script loads, package dependencies, environment variable names, webhook handlers, database fields, and tests.
   - Identify PayPal IDs in use: order, capture, authorization, refund, subscription, plan, product, payout batch/item, dispute, webhook event, invoice/custom/reference IDs.
   - Record unknowns instead of assuming them.

2. **Classify**
   - Determine whether the system uses SDK v6, older JS SDK/Smart Buttons, Orders v2, Payments v2, supported REST v1 product APIs, REST Payments v1, NVP/SOAP, IPN, PDT, or a third-party wrapper.
   - Classify each flow as current/supported, legacy-maintenance, deprecated, or unknown/verify based on supplied context or official PayPal documentation.

3. **Select architecture**
   - Pick one primary path per flow.
   - Keep browser approval/session code separate from server money movement and reconciliation.
   - Add or preserve webhooks for asynchronous state changes.
   - Include migration compatibility only where required by existing data or traffic.

4. **Plan edits**
   - State the minimal file and API changes required.
   - Preserve existing framework conventions, route style, error handling, logging, dependency policy, and test structure.
   - Avoid broad rewrites unless the existing design prevents a correct PayPal lifecycle.

5. **Implement or specify**
   - For code generation, produce complete handlers/components for the selected framework where enough context exists.
   - For repository edits, modify the smallest necessary set of files and keep configuration placeholders explicit.
   - For answer-only tasks, provide implementation-ready architecture, endpoint responsibilities, state model, and test cases.

6. **Validate**
   - Run available tests, type checks, lint checks, route tests, or local build commands when tools are available and safe.
   - Verify sandbox flow coverage: create order, approve, capture/authorize, webhook delivery, duplicate callback, duplicate webhook, cancel, refund, subscription lifecycle, payout reconciliation, or dispute flow as applicable.
   - If execution is not possible, provide a verification checklist and list untested assumptions.

7. **Report**
   - Summarize the selected PayPal path, files changed or recommended, validation performed, residual risks, and missing production checks.
   - For audits, use severity-labeled findings and exact required changes.

## Integration map

| Scenario | Default path | Checks |
|---|---|---|
| One-time checkout | JavaScript SDK v6 browser session + Orders API v2 create/capture | Server owns cart, amount, currency, capture, and fulfillment. |
| Authorize now, capture later | Orders API v2 `AUTHORIZE` + Payments API v2 authorization/capture endpoints | Verify authorization windows, reauthorization rules, and business eligibility. |
| Refunds | Payments API v2 capture refund endpoint | Refund by capture ID; enforce authorization and partial-refund rules. |
| Subscriptions | Catalog Products, Billing Plans, Billing Subscriptions v1 + webhooks | Grant access only from verified subscription state. |
| Payouts | Payouts API v1 | Use stable sender batch IDs and item-level reconciliation. |
| Webhooks | Webhooks Management/Verification v1 | Verify signature, environment, event ID deduplication, and internal state transition. |
| Disputes | Customer Disputes API v1 | Model dispute state separately from order/payment state. |
| Identity, client tokens, vaulting, Fastlane, card fields, wallets | Product-specific SDK v6 components and REST token/setup-token flows | Verify account approval, region/currency support, and PCI implications. |
| Legacy maintenance | REST Payments v1, NVP/SOAP, older Smart Buttons, IPN, PDT | Preserve existing behavior while planning migration to current equivalents. |

## Reference routing

- Read `references/sdk-v6.md` for JavaScript SDK v6 browser setup, session flow, eligibility checks, and older Smart Buttons migration.
- Read `references/rest-apis.md` for server REST patterns, Orders v2, Payments v2, active REST v1 product APIs, OAuth, webhooks, subscriptions, payouts, refunds, and storage fields.
- Read `references/legacy-and-migration.md` for REST Payments v1, NVP/SOAP, IPN, PDT, older Smart Buttons, and migration audits.
- Read `references/implementation-checklist.md` before finalizing production code, architecture reviews, migrations, or security checks.

## Output formats

### Implementation plan

Return:

1. **Target flow:** product, SDK/API versions, runtime/framework, account role, environment
2. **Architecture:** browser responsibilities, server endpoints, webhook handler, data model
3. **Implementation steps:** ordered file/API changes
4. **Validation:** tests or sandbox checks to run
5. **Risks/unknowns:** production checks requiring confirmation

### Code change summary

Return:

1. **Changed files:** concise list
2. **PayPal path:** selected SDK/API surface
3. **Behavior:** lifecycle implemented or changed
4. **Validation:** commands run and results, or unrun checks
5. **Remaining work:** required secrets, dashboard setup, webhook registration, product approvals, live-mode checks

### Audit result

Return:

1. **Verdict:** compatible / partially compatible / incompatible / insufficient information
2. **Target integration:** product, SDK/API version, framework/runtime, account role, environment
3. **Findings:** severity-labeled issues
4. **Required changes:** specific API, design, or code changes
5. **Legacy/deprecation notes:** status and migration implications
6. **Missing information:** details needed for a reliable conclusion
