# JavaScript SDK v6

Use this reference when the task involves PayPal JavaScript SDK v6 browser code, SDK compatibility review, or migration from older Smart Buttons / SDK v5 patterns. First inspect the existing script loader, component initialization, browser callbacks, and server endpoints before proposing SDK v6 changes.

## Scope

JavaScript SDK v6 is the browser layer for PayPal UI, eligibility checks, custom elements, and payment sessions. It does not replace server-side REST calls for order creation, capture, authorization, refund, subscription management, payout creation, or webhook verification.

## Default browser architecture

1. Load the v6 core script on pages that need PayPal payment UI.
2. Initialize the SDK with `window.paypal.createInstance(...)`.
3. Use a browser-safe `clientId` for most integrations.
4. Use a server-generated `clientToken` only for flows that require it, such as Fastlane or eligible vaulting flows.
5. Check funding/payment eligibility before showing payment buttons.
6. Create a payment session, such as `createPayPalOneTimePaymentSession(...)` for PayPal checkout.
7. On button click, call `session.start(...)` and pass a server-order creation promise that resolves to `{ orderId: "..." }`.
8. In `onApprove`, call the server to capture or authorize the order.
9. Reconcile final state through verified webhooks and stored PayPal IDs.

## Script loading

Use the v6 core script, not the v5 query-string SDK script.

```html
<!-- Live -->
<script src="https://www.paypal.com/web-sdk/v6/core"></script>

<!-- Sandbox -->
<script src="https://www.sandbox.paypal.com/web-sdk/v6/core"></script>
```

Do not use this as a v6 example:

```html
<script src="https://www.paypal.com/sdk/js?client-id=CLIENT_ID&currency=USD"></script>
```

That is the older SDK loading style.

## SDK initialization

For most direct merchant checkout integrations:

```javascript
const sdkInstance = await window.paypal.createInstance({
  clientId: "PAYPAL_CLIENT_ID",
  components: ["paypal-payments"],
  pageType: "checkout"
});
```

For partner integrations, include the seller merchant identifier when required:

```javascript
const sdkInstance = await window.paypal.createInstance({
  clientId: "PARTNER_CLIENT_ID",
  merchantId: "SELLER_MERCHANT_ID",
  components: ["paypal-payments"],
  pageType: "checkout"
});
```

For Fastlane or supported vaulting flows that require a client token:

```javascript
const { accessToken: clientToken } = await fetch("/api/paypal/browser-client-token", {
  credentials: "include"
}).then((response) => response.json());

const sdkInstance = await window.paypal.createInstance({
  clientToken,
  components: ["fastlane"],
  pageType: "checkout"
});
```

The server must generate the client token. Do not expose the client secret or a server OAuth access token to browser code.

## Eligibility checks

Use eligibility checks before showing payment UI. Eligibility depends on buyer context, currency, country, merchant account, device, and enabled PayPal products.

```javascript
const paymentMethods = await sdkInstance.findEligibleMethods({
  currencyCode: "USD"
});

if (paymentMethods.isEligible("paypal")) {
  document.querySelector("paypal-button").hidden = false;
}
```

Verify exact method names for Pay Later, Venmo, Apple Pay, Google Pay, card fields, Fastlane, and regional funding sources against the official documentation for the enabled component.

## One-time PayPal payment session

Use this integration shape for PayPal checkout. Keep order creation and capture on the server.

```html
<paypal-button id="paypal-button" type="pay" hidden></paypal-button>

<script src="https://www.sandbox.paypal.com/web-sdk/v6/core"></script>
<script>
  async function postJson(url, body) {
    const response = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify(body || {})
    });

    const payload = await response.json().catch(() => ({}));
    if (!response.ok) {
      throw new Error(payload.message || `Request failed: ${response.status}`);
    }
    return payload;
  }

  async function createOrder() {
    const order = await postJson("/api/paypal/orders", { cartId: window.currentCartId });
    return { orderId: order.id || order.orderId };
  }

  async function onApprove({ orderId }) {
    const result = await postJson(`/api/paypal/orders/${encodeURIComponent(orderId)}/capture`);
    if (result.status !== "COMPLETED") {
      throw new Error(`Unexpected PayPal capture status: ${result.status}`);
    }
    window.location.assign(`/checkout/success?orderId=${encodeURIComponent(orderId)}`);
  }

  async function onPayPalLoaded() {
    const sdkInstance = await window.paypal.createInstance({
      clientId: "PAYPAL_CLIENT_ID",
      components: ["paypal-payments"],
      pageType: "checkout"
    });

    const paymentMethods = await sdkInstance.findEligibleMethods({ currencyCode: "USD" });
    if (!paymentMethods.isEligible("paypal")) return;

    const paypalButton = document.querySelector("paypal-button");
    paypalButton.hidden = false;

    const paypalSession = sdkInstance.createPayPalOneTimePaymentSession({
      onApprove,
      onCancel(data) {
        window.location.assign("/checkout/cancelled");
      },
      onError(error) {
        console.error("PayPal checkout error", error);
        alert("PayPal checkout could not be completed. Please try again.");
      }
    });

    paypalButton.addEventListener("click", async () => {
      await paypalSession.start({ presentationMode: "auto" }, createOrder());
    });
  }

  window.addEventListener("load", onPayPalLoaded);
</script>
```

## Required server endpoints for this browser flow

| Endpoint | Responsibility |
|---|---|
| `POST /api/paypal/orders` | Validate cart/session server-side, create an Orders API v2 order, persist internal order state, return the PayPal order ID. |
| `POST /api/paypal/orders/{orderId}/capture` | Verify the order belongs to the authenticated checkout session, capture through Orders API v2, verify amount/currency/status, persist capture ID. |
| `POST /api/paypal/webhooks` | Verify webhook signatures, deduplicate event IDs, and reconcile order/payment/refund/dispute/subscription state. |
| `GET` or `POST /api/paypal/browser-client-token` | Generate a browser-safe client token only when the selected SDK component requires one. |

## Migration from older Smart Buttons or SDK v5

Older pattern:

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

SDK v6 pattern:

```javascript
const sdkInstance = await window.paypal.createInstance({
  clientId: "PAYPAL_CLIENT_ID",
  components: ["paypal-payments"],
  pageType: "checkout"
});

const session = sdkInstance.createPayPalOneTimePaymentSession({ onApprove, onCancel, onError });
await session.start({ presentationMode: "auto" }, createOrder());
```

Migration requirements:

- Replace `paypal.Buttons(...).render(...)` with SDK v6 custom elements and event listeners.
- Move callbacks from button configuration to payment sessions.
- Replace SDK v5 script URL configuration with `window.paypal.createInstance(...)` options.
- Ensure `createOrder()` returns a promise resolving to `{ orderId: "..." }`.
- Capture or authorize on the server, not from browser-side `actions.order.capture()`.
- Keep legacy Smart Buttons examples only for SDK v5 or older maintenance tasks.

## Component-specific guidance

| Component / flow | Guidance |
|---|---|
| PayPal and Pay Later checkout | Use `paypal-payments`, eligibility checks, payment sessions, and server-created Orders API v2 orders. |
| Venmo | Enable the Venmo component only where available and verify US-specific support and eligibility. |
| Card fields | Use PayPal-hosted/card-field components; do not collect raw card data on the merchant server unless the user explicitly accepts PCI scope. |
| Fastlane | Expect client-token authentication and account/product eligibility. |
| Apple Pay / Google Pay | Verify domain, wallet, browser, device, region, and merchant eligibility requirements. |
| Vaulting / saved payment methods | Expect client-token or setup-token/payment-token flows and additional account approval. |
| Partner/platform flows | Include partner attribution and seller merchant context where required; do not assume direct-merchant checkout semantics. |

## Common SDK v6 review findings

- Uses `paypal.Buttons(...)` while claiming SDK v6 compatibility.
- Loads `https://www.paypal.com/sdk/js?...` instead of the v6 core script.
- Passes amounts from browser code into the order creation request.
- Captures through browser-side `actions.order.capture()`.
- Treats return URL or `onApprove` as final payment success without server capture verification.
- Omits eligibility checks and fallback UI.
- Uses a client token for all flows when a client ID is sufficient.
- Exposes client secret or OAuth access token to the browser.
