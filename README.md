# JOVEpay Webview Checkout SDK

In-page payment and donation checkout for merchant sites. The SDK opens the
JOVEpay portal widgets (`/pay/embeds/...`) inside a sandboxed iframe and relays
lifecycle events to your page via `postMessage`.

This package ships two consumption styles:

| Style | How |
| --- | --- |
| **Vanilla HTML** | Load a single `index.min.js`, then `new JovePayCheckout({…}).open()` |
| **TypeScript / monorepo** | `import { JovePayCheckout } from '@aegis/jovepay-webpay'` |

---

## Quick start (vanilla HTML)

### 1. Add the script

```html
<script src="https://cdn.jsdelivr.net/gh/livialink/jovepay-webpay@main/index.min.js"></script>
```

Local / self-hosted:

```html
<script src="/path/to/index.min.js"></script>
```

After the script loads, `window.JovePayCheckout` is available.

### 2. Initialize and open

```html
<button id="pay">Pay now</button>

<script src="https://cdn.jsdelivr.net/gh/livialink/jovepay-webpay@main/index.min.js"></script>
<script>
  const button = document.getElementById('pay');

  button.addEventListener('click', function () {
    const checkout = new JovePayCheckout({
      mode: 'payment',
      invoiceId: 'YOUR_INVOICE_ID',
      theme: 'light',
      locale: 'en',
      onSuccess: function (payload) {
        console.log('Payment succeeded', payload);
      },
      onError: function (payload) {
        console.error('Payment failed', payload);
      },
      onClose: function (reason) {
        console.log('Closed:', reason);
      },
    });

    checkout.open();
  });
</script>
```

Equivalent factory form (no `new` required):

```js
const checkout = JovePayCheckout.create({ /* same options */ });
checkout.open();
```

---

## Payment modes

### A. Invoice checkout (`?iid=`)

Use when you already created an invoice in JOVEpay (dashboard / API / diuto).

```js
new JovePayCheckout({
  mode: 'payment',
  invoiceId: '1598462043',
  theme: 'light',
  locale: 'en',
  isTestnet: false,
  onSuccess: (payload) => console.log(payload),
}).open();
```

Loads:

`https://www.jovepay.com/pay/embeds/payment-widget?iid=…&theme=light&locale=en`

### B. CMS / order checkout (`?params=`)

Use from a storefront when you have an order total and merchant API key.
This matches the portal `paymentPageBuilder` / WooCommerce payload shape.

**Required:** `apiKey`, `orderId`, `priceAmount`, `priceCurrency`  
**Optional:** `successUrl`, `cancelUrl`, customer + line-item metadata  
**Not allowed:** `ipnCallbackUrl` (configure IPN on the server / dashboard)

```js
new JovePayCheckout({
  mode: 'payment',
  apiKey: 'YOUR_API_KEY',
  orderId: '1001',
  priceAmount: '49.99',
  priceCurrency: 'USD',
  successUrl: 'https://merchant.example/checkout/success', // optional
  cancelUrl: 'https://merchant.example/checkout/cancel',   // optional
  customerName: 'Ada Lovelace',
  customerEmail: 'ada@example.com',
  products: [{ name: 'Widget', quantity: 1, price: '49.99' }],
  dataSource: 'custom',
  theme: 'dark',
  locale: 'en',
  onSuccess: (payload) => {
    // Redirect yourself if you prefer not to rely on successUrl inside the widget
    window.location.href = '/thanks';
  },
}).open();
```

The SDK base64-encodes the payload into the `params` query string (same encoding
CMS plugins use). `ipnCallbackUrl` is rejected at type-check and runtime.

### C. Donation widget

```js
new JovePayCheckout({
  mode: 'donation',
  apiKey: 'YOUR_API_KEY',
  theme: 'light',
  locale: 'en',
}).open();
```

Loads:

`https://www.jovepay.com/pay/embeds/donation-widget?apiKey=…`

---

## Inline embed (no modal)

Mount into a page slot instead of a fullscreen overlay:

```html
<div id="pay-slot" style="min-height: 640px"></div>

<script>
  JovePayCheckout.create({
    mode: 'payment',
    invoiceId: 'YOUR_INVOICE_ID',
    display: 'inline',
  }).mount('#pay-slot');
</script>
```

---

## Options reference

### Shared

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `mode` | `'payment' \| 'donation'` | — | Required |
| `env` | `'production' \| 'development'` | `'production'` | Portal host (`https://www.jovepay.com` vs `http://localhost:3000`) |
| `baseUrl` | `string` | — | Override portal origin (wins over `env`) |
| `theme` | `'light' \| 'dark'` | `'light'` | Passed to the widget |
| `locale` | `string` | `'en'` | Widget locale (`en`, `es`, `fr`, …) |
| `isTestnet` | `boolean` | `false` | Use testnet networks |
| `paymentId` | `string` | — | Resume payment (`pid`) |
| `display` | `'modal' \| 'inline'` | `'modal'` | Overlay vs mount target |
| `autoCloseOnSuccess` | `boolean` | `true` (modal) | Close after success |
| `autoCloseOnError` | `boolean` | `false` | Close after failure |
| `closeOnBackdrop` | `boolean` | `true` | Click dimmed backdrop to close |
| `closeOnEscape` | `boolean` | `true` | Escape key closes modal |
| `query` | `object` | `{}` | Extra query params |

### Callbacks

| Callback | Signature | When |
| --- | --- | --- |
| `onReady` | `() => void` | Iframe signaled `WIDGET_READY` (or load fallback) |
| `onInit` | `(payload) => void` | `PAYMENT_INIT` |
| `onSuccess` | `(payload) => void` | `PAYMENT_SUCCESS` / `FINISHED` / `CONFIRMED` |
| `onError` | `(payload) => void` | `PAYMENT_FAILED` / `EXPIRED` |
| `onClose` | `(reason, payload?) => void` | Widget torn down |
| `onEvent` | `(event) => void` | Every typed widget event |

`onClose` reasons: `'user' | 'success' | 'error' | 'api' | 'destroy' | 'superseded'`.

### Instance methods

```js
checkout.open();           // open modal
checkout.mount('#el');     // inline mount
checkout.close();          // close + cleanup
checkout.destroy();        // alias of close('destroy')
checkout.getStatus();      // 'idle' | 'opening' | 'open' | 'closing' | 'closed'
checkout.getEmbedUrl();    // resolved iframe URL
```

---

## postMessage protocol

The portal posts to the parent only when running inside an iframe:

```js
{
  source: 'jovepay-widget',
  event: {
    name: 'WIDGET_READY' | 'WIDGET_CLOSE' | 'PAYMENT_INIT' | 'PAYMENT_SUCCESS' | 'PAYMENT_FAILED' | …,
    payload: { /* payment fields */ }
  }
}
```

The SDK verifies `event.origin` against the configured portal base URL before
dispatching callbacks. Event name constants are also on the class:

```js
JovePayCheckout.EVENTS.PAYMENT_SUCCESS; // "PAYMENT_SUCCESS"
```

---

## Development & build

From the monorepo root (or this package):

```bash
# install (once from repo root)
pnpm install

# build browser bundle → libs/jovepay-webpay/dist/index.min.js
pnpm --filter @aegis/jovepay-webpay build

# watch mode
pnpm --filter @aegis/jovepay-webpay build:watch
```

Build output (`dist/`):

```
dist/
  index.min.js      # browser IIFE (window.JovePayCheckout)
  index.min.js.map
  index.html        # vanilla demo page
  README.md
  package.json      # CDN package metadata
```

Open the demo:

```bash
pnpm --filter @aegis/jovepay-webpay demo
# → http://localhost:4173
```

Or open `dist/index.html` directly after `pnpm build` (some browsers restrict
`file://` modules; prefer the demo server).

### TypeScript consumers in this monorepo

```ts
import { JovePayCheckout } from '@aegis/jovepay-webpay';

const checkout = new JovePayCheckout({
  mode: 'payment',
  invoiceId: '…',
});
checkout.open();
```

Add `"@aegis/jovepay-webpay": "workspace:*"` to the app `package.json` if needed.
Path alias `@aegis/jovepay-webpay` is already in `tsconfig.base.json`.

---

## CDN publish (deployment)

On every push to `main` that touches `libs/jovepay-webpay/**`, GitHub Actions:

1. Installs dependencies
2. Runs `pnpm --filter @aegis/jovepay-webpay build`
3. Publishes the contents of `libs/jovepay-webpay/dist/` to the
   **`livialink/jovepay-pay`** repository (`main` branch)

After publish, the public script URL is:

```text
https://cdn.jsdelivr.net/gh/livialink/jovepay-webpay@main/index.min.js
```

Pin a commit for production if you need immutable caching:

```text
https://cdn.jsdelivr.net/gh/livialink/jovepay-webpay@main/index.min.js
```

---

## Security notes

- Iframe `sandbox` allows forms, scripts, same-origin, popups, and user-activated top navigation.
- Parent only accepts messages whose `origin` matches the configured portal host.
- Never put secret API credentials that can mutate payouts into client-side config; the public `apiKey` used by widgets is the merchant publishable key.
- Do not pass `ipnCallbackUrl` through the SDK — IPN must be configured server-side.

