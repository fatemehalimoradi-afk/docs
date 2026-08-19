# Knit Bundles — Headless Integration Guide

Knit Bundles exposes public endpoints for **headless storefronts, mobile apps,
and custom integrations** that don't run on the standard Shopify Liquid theme.

There are two ways to integrate, depending on how much control you want:

| Mode | You get | You build |
| --- | --- | --- |
| **Embed** (fastest) | A ready-to-use iframe that renders the full bundle widget and handles all selection logic | An `<iframe>` + a small `postMessage` handler that forwards add-to-cart to your native cart |
| **Data API** (full control) | Normalized bundle JSON (products, tiers, pricing, discounts) | Your own UI + your own cart logic |

Both modes are served from the **Knit app origin**, not from your `myshopify.com`
storefront. This is deliberate — see [Why the app origin?](#why-the-app-origin)
below.

---

## Base URL

```
https://app.knit-bundle.co
```

All headless endpoints live under `/headless/*`. Every request identifies the
target store with a `shop` query parameter:

```
?shop={your-store}.myshopify.com
```

---

## Prerequisites

1. Install the **Knit Bundles** app on your Shopify store.
2. Configure **at least one active bundle**.
3. Open the app once in the Shopify admin — this provisions the Storefront
   access token the headless endpoints need. (Until this happens, the endpoints
   return `APP_NOT_INSTALLED`.)

You do **not** need to embed the theme app extension for the headless endpoints
to work.

---

## Endpoint 1 — Bundle Data

```
GET https://app.knit-bundle.co/headless/bundles?shop={shop}&product_id={product_id}
```

Returns all active bundles for the shop as **fully normalized JSON**: resolved
products, typed conditions and rewards, and the assets (JS/CSS) associated with
each bundle. Use this when you want to build your own bundle UI.

### Query parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | Your `{store}.myshopify.com` domain. |
| `product_id` | No | Numeric Shopify product ID. When provided, the response is **filtered to only the bundles applicable to that product** (product-single match, or collection membership). Omit it to receive every bundle. |
| `handle` | No | Return only the bundle with this metaobject handle. |
| `type` | No | Filter bundles by type (defaults to the standard bundle type). |
| `language` | No | Storefront language code (e.g. `EN`, `DE`) for `@inContext` translation and market pricing. |
| `country` | No | Storefront country code (e.g. `US`, `DE`) for market pricing. |

### Example

```
GET https://app.knit-bundle.co/headless/bundles?shop=dyvan-dev.myshopify.com&product_id=7494254788801
```

### Response

```json
{
  "ok": true,
  "bundles": [
    {
      "id": "72",
      "version": 3,
      "title": "Buy More - Save More",
      "subtitle": null,
      "atcOverride": false,
      "handle": "bundle-72",
      "type": "$app:bundle",
      "isActive": true,
      "pageProductId": "7494254788801",
      "selectors": [
        {
          "kind": "productSingle",
          "id": "157",
          "product": {
            "id": "7494254788801",
            "handle": "filter-shot-contour-pact",
            "title": "16 Filter Shot Contour Pact",
            "variants": [
              {
                "id": "42299900362945",
                "title": "Peach | 7g",
                "priceAmount": 51,
                "currencyCode": "EUR",
                "available": true,
                "inventoryQuantity": 12,
                "image": {
                  "url": "https://cdn.shopify.com/s/files/.../image.jpg",
                  "altText": "16 Filter Shot Contour Pact",
                  "width": 1000,
                  "height": 1000
                }
              }
            ]
          }
        }
      ],
      "conditionSets": [
        {
          "id": "213",
          "conditions": [
            {
              "id": "1469",
              "metaobjectType": "app--372708343809--condition",
              "satisfiesFor": [{ "selectorId": "157" }],
              "type": "quantity",
              "minimum": 2,
              "maximum": 2
            }
          ],
          "rewardSet": {
            "id": "213",
            "rewards": [
              {
                "id": "1277",
                "appliesTo": [{ "selectorId": "157" }],
                "type": "percentageDiscount",
                "percentage": 19,
                "quantity": null
              }
            ]
          }
        }
      ],
      "assets": [
        { "assetId": "a1", "type": "JS", "url": "https://cdn.shopify.com/...", "content": null },
        { "assetId": "a2", "type": "CSS", "url": "https://cdn.shopify.com/...", "content": null }
      ]
    }
  ]
}
```

### Caching

Successful responses are sent with:

```
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

Bundle changes may take up to ~5 minutes to propagate through caches.

---

## Endpoint 2 — Embed (iframe)

```
GET https://app.knit-bundle.co/headless/embed?shop={shop}&product_id={product_id}
```

Returns a self-contained HTML page that loads the Knit widget runtime, fetches
the applicable bundles, and renders the bundle widget for the given product.
Drop it into an `<iframe>` and you're done rendering — you only need to handle
**add-to-cart** (see [postMessage bridge](#postmessage-bridge)).

### Query parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | Your `{store}.myshopify.com` domain. |
| `product_id` | Recommended | Numeric Shopify product ID of the current page. Determines which bundles render. |
| `variant_id` | No | Numeric variant ID for the initially selected variant. |
| `handle` | No | Render only a specific bundle handle. |
| `language` | No | Language code for the rendered page (`lang` attribute + translations). |

### Usage

```html
<iframe
  id="knit-bundle"
  src="https://app.knit-bundle.co/headless/embed?shop=dyvan-dev.myshopify.com&product_id=7494254788801"
  style="width:100%;border:0;"
  scrolling="no"
></iframe>

<script>
  // See the postMessage bridge section below for the full handler.
</script>
```

The embed page is served with `Cache-Control: no-store` and **without**
`X-Frame-Options` / `frame-ancestors`, so it is framable from any origin.

---

## postMessage bridge

The embed iframe communicates with your host page over `window.postMessage`.
It **renders the widget and manages selection state itself**, but it cannot add
to your cart — that's your job. You must listen for the messages below.

Every message includes `source`, `v` (version), and `type`. Ignore any message
whose `source`/`v` you don't recognize.

### Messages from iframe → your page

`source: "nameless-bundle"`, `v: 1`

| `type` | Payload | Meaning |
| --- | --- | --- |
| `ready` | — | Widget mounted and ready. |
| `resize` | `{ height: number }` | New content height in px — resize the iframe element to avoid inner scrollbars. |
| `addToCart` | `{ requestId, lineItems }` | The shopper triggered add-to-cart. Add the line items to your cart, then reply. |

### Messages from your page → iframe

`source: "nameless-bundle-host"`, `v: 1`

| `type` | Payload | Meaning |
| --- | --- | --- |
| `addToCartResult` | `{ requestId, ok, code?, message? }` | Your response to an `addToCart`. Echo the same `requestId`. Set `ok: false` with an optional `code`/`message` on failure. |

### `lineItems` shape

```js
[
  {
    variantId: 42299900362945,   // numeric Shopify variant ID
    quantity: 2,
    attributes: [                // Shopify line-item properties, key/value pairs
      { key: "__FB_ATC_UID", value: "72|3|213|157|1620000000000" }
    ]
  }
]
```

> **Critical:** the `__FB_ATC_UID` attribute must reach Shopify as a **line-item
> property**. The Knit Discount Function uses it to group bundle lines and apply
> the tier discount. If your cart layer strips custom line-item properties, the
> discount will not apply.

### Minimal host handler

```html
<script>
  const IFRAME = document.getElementById("knit-bundle");
  const IFRAME_SOURCE = "nameless-bundle";
  const HOST_SOURCE = "nameless-bundle-host";
  const VERSION = 1;

  window.addEventListener("message", async (event) => {
    const msg = event.data;
    if (!msg || msg.source !== IFRAME_SOURCE || msg.v !== VERSION) return;

    switch (msg.type) {
      case "ready":
        break;

      case "resize":
        IFRAME.style.height = msg.height + "px";
        break;

      case "addToCart": {
        try {
          // Add msg.lineItems to YOUR cart, preserving `attributes`
          // as Shopify line-item properties.
          await myCart.add(msg.lineItems);
          reply(msg.requestId, true);
        } catch (err) {
          reply(msg.requestId, false, "CART_REJECTED", String(err));
        }
        break;
      }
    }
  });

  function reply(requestId, ok, code, message) {
    IFRAME.contentWindow.postMessage(
      { source: HOST_SOURCE, v: VERSION, type: "addToCartResult", requestId, ok, code, message },
      "*"
    );
  }
</script>
```

If you don't reply to an `addToCart` within ~10 seconds, the widget resolves the
action as failed (`CART_TIMEOUT`) and shows an error to the shopper.

---

## Response schema reference

The `/headless/bundles` endpoint returns `{ ok: true, bundles: Bundle[] }`.
Each `Bundle` has this shape (TypeScript):

```typescript
interface Bundle {
  id: string;                 // numeric bundle id, as string
  version: number;            // bumps on every merchant edit
  title: string;
  subtitle: string | null;
  atcOverride: boolean;       // widget replaces the native ATC button when true
  handle: string | null;
  type: string;
  isActive: boolean;
  pageProductId?: string;     // present when product_id was supplied
  selectors: Selector[];
  conditionSets: ConditionSet[];
  assets: Asset[];
}

// One of three kinds
type Selector =
  | { kind: "productSingle"; id: string; product: Product }
  | { kind: "collectionSingle"; id: string; collection: Collection; resolvedProduct: Product | null }
  | { kind: "collectionMulti"; id: string; collection: Collection };

interface Product {
  id: string;                 // numeric product id, as string (GID stripped)
  handle: string;
  title: string;
  variants: Variant[];
}

interface Variant {
  id: string;                 // numeric variant id, as string
  title: string;
  priceAmount: number;
  currencyCode: string;
  available: boolean;
  inventoryQuantity: number | null;
  image: { url: string; altText: string | null; width: number | null; height: number | null } | null;
}

interface Collection {
  id: string;
  handle: string;
  title: string;
  products: Product[];        // populated for collectionMulti
}

interface ConditionSet {
  id: string;
  conditions: Condition[];
  rewardSet: { id: string; rewards: Reward[] };
}

// Quantity is the only typed condition today; unrecognized ones arrive as "unknown".
type Condition =
  | { id: string; metaobjectType: string; satisfiesFor: { selectorId: string }[]; type: "quantity"; minimum?: number; maximum?: number }
  | { id: string; metaobjectType: string; satisfiesFor: { selectorId: string }[]; type: "unknown"; raw: Record<string, unknown> };

type Reward =
  | { id: string; appliesTo: { selectorId: string }[]; type: "percentageDiscount"; percentage: number; quantity: number | null }
  | { id: string; appliesTo: { selectorId: string }[]; type: "fixedDiscount"; amount: number; mode: "TOTAL" | "PER_SELECTOR" | "PER_UNIT" }
  | { id: string; appliesTo: { selectorId: string }[]; type: "unknown"; rawType: string; raw: Record<string, unknown> };

interface Asset {
  assetId: string | null;
  type: "JS" | "CSS";
  url: string | null;
  content: string | null;
}
```

### How pricing works

Bundle discounts are expressed as **conditions → rewards**, not as precomputed
prices:

- A `conditionSet` fires when its `conditions` are met for the referenced
  selector(s) — e.g. a `quantity` condition with `minimum: 2, maximum: 2` fires
  when the shopper picks quantity 2.
- The matching `rewardSet.rewards` then apply — e.g. `percentageDiscount` of 19%
  off, or a `fixedDiscount` amount.

To display "Buy 2, save 19%" tiers, map each `conditionSet` to its exact
quantity (`minimum === maximum`) and read the reward for that selector. See the
example renderer for a complete implementation pattern.

---

## Error responses

Errors return `{ ok: false, code, message }` with an HTTP status:

| HTTP | `code` | Meaning |
| --- | --- | --- |
| 400 | `INVALID_PARAMS` | `shop` is missing or not a valid `*.myshopify.com` domain. |
| 404 | `NOT_FOUND` | The headless endpoints are not enabled for this deployment. |
| 409 | `APP_NOT_INSTALLED` | The shop has no provisioned Storefront token — open the Knit app once in the Shopify admin. |
| 502 | `UPSTREAM_ERROR` | The Shopify Storefront API call failed. |

The `/headless/embed` endpoint returns plain-text `400` / `404` bodies for the
same missing-param / not-enabled conditions.

---

## Why the app origin?

Headless clients can't frame `{shop}.myshopify.com` directly: Shopify sends
`Content-Security-Policy: frame-ancestors 'none'` on every storefront response,
and it isn't merchant-configurable. So the embed page is served from the Knit
app origin, which sets no such header and is therefore framable anywhere.

Because the page lives on the app origin (not the storefront), the widget fetches
its data from `/headless/bundles` (same-origin, no CORS) rather than the storefront
App Proxy.

### Note on the App Proxy endpoints

You may see references to `https://{shop}.myshopify.com/apps/proxy/bundles` and
`/apps/proxy/embed`. Those are HMAC-signed App Proxy endpoints intended for hosts
that **can** frame the storefront origin. They return the **same normalized
`Bundle[]` schema** documented above. For headless apps and mobile/WebView hosts,
use the `/headless/*` endpoints in this guide instead.

---

## Support

Include your `shop` domain, the endpoint URL, and the full JSON error response
(or the `[nameless]` console logs from the embed iframe) when contacting support.
