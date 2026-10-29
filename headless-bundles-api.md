---
title: "Headless Bundles API"
description: "Embed the Knit widget or read bundle data on a headless storefront or mobile app"
---

Knit exposes public endpoints for **headless storefronts, mobile apps, and custom integrations** that don't run on a Shopify Liquid theme.

There are two ways to integrate:

| Mode | You get | You build |
| --- | --- | --- |
| **Embed** (fastest) | An iframe that renders the bundle widget and runs all selection and pricing logic | An `<iframe>` and a small `postMessage` handler that forwards add-to-cart to your cart |
| **Data API** (full control) | Normalized bundle JSON: products, offers, tiers, conditions, rewards | Your own UI, your own pricing display, and your own cart logic — including the discount tag |

Both are served from the **Knit app origin**, not from your `myshopify.com` storefront. See [Why the app origin?](#why-the-app-origin)

---

## Base URL

```
https://app.knit-bundle.co
```

All headless endpoints live under `/headless/*`. Every request names the store with a `shop` query parameter:

```
?shop={your-store}.myshopify.com
```

---

## Prerequisites

1. Install Knit. See [Getting started](/getting-started).
2. Create at least one **Active** bundle.
3. Open the app once in Shopify admin. This provisions the Storefront access token the headless endpoints need. Until then, they return `APP_NOT_INSTALLED`.

You do **not** need the theme app embed for the headless endpoints.

---

## Endpoint 1 — Bundle data

```
GET https://app.knit-bundle.co/headless/bundles?shop={shop}&product_id={product_id}
```

Returns the shop's bundles as normalized JSON: resolved products, offers with their tiers, typed conditions and rewards, and each bundle's display settings. Use it to build your own bundle UI.

### Query parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | Your `{store}.myshopify.com` domain |
| `product_id` | No | Shopify product ID (numeric or GID). Returns only the bundles that apply to that product — a product match, collection membership, or the bundle's own [Bundle as Product](/shopify-integration#bundle-as-product) page. Omit it to get every bundle |
| `handle` | No | Return only the bundle with this handle |
| `type` | No | Omit. The default is the current bundle type |
| `language` | No | Storefront language code (`EN`, `DE`, `PT-BR`) for translated titles and `@inContext` pricing. Also limits `translations` to that language |
| `country` | No | Storefront country code (`US`, `DE`) for market pricing |

### Example

```
GET https://app.knit-bundle.co/headless/bundles?shop=your-store.myshopify.com&product_id=7494254788801
```

```json
{
  "ok": true,
  "primaryLocale": "en",
  "bundles": [
    {
      "id": "72",
      "version": 3,
      "title": "Buy More - Save More",
      "handle": "bundle-72",
      "isActive": true,
      "atcOverride": false,
      "subscriptionsEnabled": false,
      "currencyCode": "EUR",
      "pageProductId": "7494254788801",
      "selectors": [
        {
          "kind": "productSingle",
          "type": "multi",
          "itemType": "product",
          "grouping": "variant",
          "id": "157",
          "excludedVariantIds": [],
          "product": {
            "id": "7494254788801",
            "handle": "filter-shot-contour-pact",
            "title": "16 Filter Shot Contour Pact",
            "options": [{ "name": "Shade", "values": ["Peach | 7g"] }],
            "requiresSellingPlan": false,
            "sellingPlanGroups": [],
            "variants": [
              {
                "id": "42299900362945",
                "title": "Peach | 7g",
                "selectedOptions": [{ "name": "Shade", "value": "Peach | 7g" }],
                "priceAmount": 51,
                "currencyCode": "EUR",
                "available": true,
                "inventoryQuantity": 12,
                "image": {
                  "url": "https://cdn.shopify.com/s/files/.../image.jpg",
                  "altText": "16 Filter Shot Contour Pact",
                  "width": 1000,
                  "height": 1000
                },
                "sellingPlanAllocations": []
              }
            ]
          }
        }
      ],
      "offers": [
        {
          "id": "88",
          "conditionClauses": [],
          "tiers": [
            {
              "id": "213",
              "conditionClauses": [
                {
                  "id": "301",
                  "conditions": [
                    {
                      "id": "1469",
                      "metaobjectType": "quantity",
                      "type": "quantity",
                      "satisfiesFor": [{ "selectorId": "157" }],
                      "min": 2,
                      "max": 2
                    }
                  ]
                }
              ],
              "rewards": [
                {
                  "id": "1277",
                  "type": "percentageDiscount",
                  "target": "PRODUCT",
                  "appliesTo": [{ "selectorId": "157" }],
                  "percentage": 19,
                  "quantity": null
                }
              ]
            }
          ]
        }
      ],
      "pdpDisplay": {
        "id": "55",
        "isActive": "true",
        "priority": "1",
        "isPopup": "false",
        "atcOverride": "false",
        "injectSelector": "",
        "relativePosition": "AFTER",
        "hideSelectors": [],
        "jsUrl": "https://…/renderer.js",
        "cssUrl": "https://…/styling.css",
        "params": {},
        "customCss": "",
        "styleVars": null,
        "assets": []
      },
      "assets": []
    }
  ]
}
```

The example is trimmed: the bundle's `type` field and the optional top-level `shop` object are left out.

### Response notes

- **Bundles are not filtered by status.** Skip a bundle whose `isActive` is false, and one whose `pdpDisplay.isActive` is `"false"` (the merchant hid it on product pages).
- **`pdpDisplay` scalars are strings** (`"true"`, `"1"`). Read `jsUrl`/`cssUrl` first; fall back to `pdpDisplay.assets`, then the bundle-level `assets`.
- **IDs:** product and variant IDs are numeric strings (GID stripped). A collection's `id` is a full GID — pass it as is to [collection-products](#more-of-a-collections-products).
- **Excluded variants** (`excludedVariantIds`) are already removed from a selector's products.
- **A bundle with a General product set** (any product of the shop) needs a page product, so it is left out when you don't pass `product_id`.

### More of a collection's products

A collection's `products` holds its first batch, and `collection.nextCursor` says where the next batch starts (`null` when every product is here).

```
GET /headless/collection-products?shop={shop}&collection_id={collection.id}&after={collection.nextCursor}
```

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | As above |
| `collection_id` | **Yes** | `collection.id`, the GID |
| `after` | **Yes** | `collection.nextCursor` |
| `language`, `country` | No | As above |

The response is `{ ok: true, products: Product[], nextCursor }` with up to 50 products. A selector's `excludedVariantIds` are **not** removed here: drop those variants, and any product left with none, before listing them.

### Caching

Successful responses are sent with:

```
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

Bundle changes can take up to about 5 minutes to reach clients through caches.

---

## Endpoint 2 — Embed (iframe)

```
GET https://app.knit-bundle.co/headless/embed?shop={shop}&product_id={product_id}
```

Returns a self-contained HTML page that loads the Knit widget runtime, fetches the bundles that apply to the product, and renders them. Put it in an `<iframe>`; you only handle **add-to-cart** (see [postMessage bridge](#postmessage-bridge)).

### Query parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | Your `{store}.myshopify.com` domain |
| `product_id` | Recommended | Numeric product ID of the current page. Decides which bundles render |
| `language` | No | Language for the page (`lang` attribute, translations, and the widget's locale) |
| `country` | No | Country code for market pricing |

`handle`, `type`, and `variant_id` are accepted but don't currently limit which bundles render or preselect a variant.

### What the embed renders

- Every bundle that applies to the product, except those hidden on product pages and those shown as a popup.
- One-time purchases only: subscription plans are not offered in the embed.
- Always the widget's own add-to-cart button. The theme-button override doesn't apply.

### Usage

```html
<iframe
  id="knit-bundle"
  src="https://app.knit-bundle.co/headless/embed?shop=your-store.myshopify.com&product_id=7494254788801"
  style="width:100%;border:0;"
  scrolling="no"
></iframe>
```

The embed page is served with `Cache-Control: no-store` and **without** `X-Frame-Options` or `frame-ancestors`, so any origin can frame it.

---

## postMessage bridge

The embed iframe talks to your page over `window.postMessage`. It renders the widget and manages selection state itself, but it can't add to your cart — that's your job.

Every message has `source`, `v` (version), and `type`. Ignore any message whose `source` or `v` you don't recognize.

### From the iframe to your page

`source: "nameless-bundle"`, `v: 1`

| `type` | Payload | Meaning |
| --- | --- | --- |
| `ready` | — | The widget mounted |
| `resize` | `{ height: number }` | New content height in px. Apply it as is — it can shrink as well as grow |
| `addToCart` | `{ requestId, lineItems }` | The shopper pressed add-to-cart. Add the lines, then reply |

`ready` and `resize` are posted to both `window.parent` and `window.top`; `addToCart` only to `window.parent`.

### From your page to the iframe

`source: "nameless-bundle-host"`, `v: 1`

| `type` | Payload | Meaning |
| --- | --- | --- |
| `addToCartResult` | `{ requestId, ok, code?, message? }` | Your answer to an `addToCart`. Echo the same `requestId`. On failure send `ok: false` with an optional `code` and `message` |

If you don't reply within **10 seconds**, the add fails and the widget shows an error.

### `lineItems` shape

```js
[
  {
    variantId: 42299900362945, // numeric Shopify variant ID
    quantity: 2,
    attributes: [
      // Shopify line-item properties as key/value pairs
      { key: "__FB_ATC_UID", value: "72|3|213|157|1715000000000" },
    ],
  },
];
```

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
      case "resize":
        IFRAME.style.height = msg.height + "px";
        break;

      case "addToCart": {
        try {
          // Add msg.lineItems to YOUR cart, keeping `attributes`
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
      "*",
    );
  }
</script>
```

---

## The discount tag (`__FB_ATC_UID`)

Bundle discounts are applied at checkout by the **Knit discount function**, a Shopify Function. It doesn't infer bundles from products alone. It only considers lines that carry this line-item property:

| | |
| --- | --- |
| **Key** | `__FB_ATC_UID` (exact spelling, two underscores) |
| **Value** | Pipe-delimited string — see below |

If the property is missing, renamed, or stripped by your cart layer, the add succeeds but **no bundle discount appears** at checkout.

### Value format

```
<bundleId>|<version>|<tierId>|<selectorId>|<atcId>
```

| Segment | Source | Role |
| --- | --- | --- |
| `bundleId` | `bundle.id` | Which bundle definition to load |
| `version` | `bundle.version` | Recorded for reference. Detection doesn't use it |
| `tierId` | `offers[].tiers[].id` | The tier this line's add was priced at. Checkout starts from it and only walks **down** the ladder — a tier above the tag is never tried |
| `selectorId` | `selectors[].id` of **this** line | Which selector the line counts for |
| `atcId` | One id per add-to-cart (epoch milliseconds) | Groups the lines of one add into one bundle. All lines of one add share it; separate adds use separate ids. Adding N copies of a bundle at once uses `<ms>`, `<ms>-2`, … `<ms>-N` |

Use the IDs from the Data API as is (numeric strings like `"72"`, not GIDs).

### Embed mode

The iframe already puts `__FB_ATC_UID` in each line's `attributes`. Forward them unchanged:

| Your cart API | How to pass it |
| --- | --- |
| Storefront API `cartLinesAdd` / Cart API | `attributes: [{ key: "__FB_ATC_UID", value: "..." }]` |
| Ajax `/cart/add.js` | `properties: { "__FB_ATC_UID": "..." }` |
| Tapcart `cart/add` | Keep the `attributes` array on each line item |

Don't filter "hidden" or underscore-prefixed properties. Many cart SDKs drop them by default — the most common cause of "items add, discount missing".

### Data API mode (your own UI)

You set the tag yourself. The rules checkout applies:

- **Tag the highest tier that holds.** Tiers of one offer are a ladder in wire order (later tiers need and give more). Checkout tries the tagged tier and then the ones below it, so tagging a lower tier caps the discount.
- **A condition counts a pool.** The units of every selector in `satisfiesFor` count together: "2 to 5 of [A, B]" holds on 1 A + 2 B. Units above `max` are capped, not refused.
- **Clauses are AND, conditions inside a clause are OR.** A tier holds when one condition from each of its clauses, and from each of its offer's clauses, holds.
- **Several offers in one add:** each offer's chosen tier tags the lines it used; a line two offers used keeps the first offer's tag. Every other line of the add carries the first applied tier, so the whole add stays one bundle.
- **Triggers** (`isTrigger: true`) count matching units anywhere in the cart and use none of them. Leave untagged the lines of a selector that only triggers name and no reward targets — unless the add is made only of such lines. With triggers, tag each offer with the higher of the tier you priced and the tier that would hold if every trigger held.
- **Nothing holds, no trigger:** add the lines without the tag.

```js
const atcId = String(Date.now()); // shared by every line of this add
await cartLinesAdd({
  lines: [
    {
      merchandiseId: "gid://shopify/ProductVariant/42299900362945",
      quantity: 2,
      attributes: [{ key: "__FB_ATC_UID", value: `72|3|213|157|${atcId}` }],
    },
  ],
});
```

<Note>
  Knit's own widget mints tags with the same engine checkout uses. A hand-built UI approximates it. When you can, use the Embed and let it tag for you.
</Note>

### How to verify

1. Add a qualifying bundle and open the cart, or place a test order.
2. Each bundle line should show `__FB_ATC_UID` with a 5-part `|`-delimited value.
3. Confirm the Knit discount is **Active** in Shopify admin → **Discounts**, and the bundle is **Active** in Knit.

If the property is there but the discount isn't, the tagged tier probably no longer exists or the quantities no longer meet it (a stale cart after a merchant edit), or the discount was turned off.

---

## Response schema reference

`/headless/bundles` returns:

```typescript
interface BundlesResponse {
  ok: true;
  bundles: Bundle[];
  primaryLocale?: string; // the shop's default language
  shop?: { styleVars: unknown; customCss: string; translations?: unknown }; // shop-wide widget styling
}

interface Bundle {
  id: string;                    // numeric bundle id, as string
  version: number;               // bumps on every merchant edit
  title: string;                 // translated for `language` when a translation exists
  handle: string | null;
  type: string;
  isActive: boolean;             // not filtered server-side
  atcOverride: boolean;          // the theme's button adds the bundle (Liquid storefront only)
  subscriptionsEnabled: boolean;
  currencyCode: string;
  pageProductId: string | null;  // the product_id you passed, numeric
  selectors: Selector[];
  offers: Offer[];
  pdpDisplay?: PdpDisplay;       // absent when the bundle has no display
  assets: Asset[];               // legacy; prefer pdpDisplay
  translations?: { v: 1; primary?: string; locales: Record<string, Record<string, string>> };
}

interface PdpDisplay {
  id: string;
  isActive: string;              // "false" ⇒ hidden on product pages
  priority: string | null;       // lower = shown first
  isPopup: string;
  atcOverride: string;
  injectSelector: string;
  relativePosition: string | null; // "BEFORE" | "AFTER" the matched element
  hideSelectors: string[];
  jsUrl: string | null;
  cssUrl: string | null;
  params: unknown;
  customCss: string;
  styleVars: unknown;
  assets: Asset[];
}

// `kind` is how to draw the selector. `type`, `itemType` and `grouping` are what the
// discount engine reads; when `grouping` is present it wins over `type`.
type Selector = {
  id: string;
  type: "single" | "multi";
  grouping?: "variant" | "product" | "general";
  excludedVariantIds: string[];
} & (
  | { kind: "productSingle"; itemType: "product"; product: Product; variantId?: string } // variantId: a one-variant item
  | { kind: "productSingle"; itemType: "collection"; product: Product; collection: Collection } // the page product, inside the collection
  | { kind: "collectionSingle"; itemType: "collection"; collection: Collection } // pick one product
  | { kind: "collectionMulti"; itemType: "collection"; collection: Collection } // mix products
);

interface Product {
  id: string;                    // numeric, as string
  handle: string;
  title: string;
  options: { name: string; values: string[] }[];
  variants: Variant[];
  requiresSellingPlan: boolean;
  sellingPlanGroups: unknown[];
}

interface Variant {
  id: string;                    // numeric, as string
  title: string;
  selectedOptions: { name: string; value: string }[];
  priceAmount: number;
  currencyCode: string;
  available: boolean;
  inventoryQuantity: number | null; // null: stock never limits it
  image: { url: string; altText: string | null; width: number | null; height: number | null } | null;
  sellingPlanAllocations: {
    sellingPlanId: string;
    priceAmount: number;
    currencyCode: string;
    compareAtPriceAmount: number | null;
    perDeliveryPriceAmount: number | null;
  }[];
}

interface Collection {
  id: string;                    // full GID
  handle: string;
  title: string;
  products: Product[];           // the first batch; empty when drawn as the page product
  nextCursor: string | null;
}

interface Offer {
  id: string;
  conditionClauses: ConditionClause[]; // shared by every tier below
  tiers: Tier[];                       // a ladder, smallest first
}

interface Tier {
  id: string;
  conditionClauses: ConditionClause[];
  rewards: Reward[];
}

interface ConditionClause {
  id: string;
  conditions: Condition[];       // one of them has to hold
}

type Condition =
  | {
      id: string;
      metaobjectType: "quantity";
      type: "quantity";
      satisfiesFor: { selectorId: string }[]; // counted together, as one pool
      min?: number;
      max?: number;              // units above it are capped, not refused
      isTrigger?: true;          // counts the whole cart, uses no units
    }
  | {
      id: string;
      metaobjectType: string;
      type: "unknown";
      satisfiesFor: { selectorId: string }[];
      raw: Record<string, unknown>;
    };

interface RewardBase {
  id: string;
  appliesTo: { selectorId: string }[];
  target: "PRODUCT" | "SHIPPING"; // SHIPPING lowers delivery and names no lines
  slice?: { after: number; maxUnits: number | null; sort: "CHEAPEST" }; // absent: every used unit
}

type Reward =
  | (RewardBase & { type: "percentageDiscount"; percentage: number; quantity: number | null })
  | (RewardBase & { type: "fixedDiscount"; amount: number; mode: "TOTAL" | "PER_SELECTOR" | "PER_UNIT" })
  | (RewardBase & { type: "setPrice"; price: number; mode: "TOTAL" | "PER_SELECTOR" | "PER_UNIT" }) // price, not a reduction; 0 = free
  | { id: string; appliesTo: { selectorId: string }[]; type: "unknown"; rawType: string; raw: Record<string, unknown> };

interface Asset {
  assetId: string | null;
  type: string;                  // "JS" or "CSS"
  url: string;
  content: string | null;
}
```

### How pricing works

Discounts are **conditions → rewards**, not precomputed prices:

- A tier holds when every clause on it, and every clause on its offer, is met by one of the clause's conditions.
- In each offer, the highest tier that holds applies, and it uses the units its pools matched. Units above a `max` stay at full price and remain free for a later offer.
- The tier's rewards then lower the used units: `percentageDiscount`, a `fixedDiscount` amount, or a `setPrice` the lines are re-priced to. `mode` spreads a money value over the whole target (`TOTAL`), each selector (`PER_SELECTOR`), or each unit (`PER_UNIT`). A `slice` lowers only part of the units — "buy 3, get 1 free" is 100% off with `{ after: 3, maxUnits: 1 }`.
- `amount` and `price` are in the shop's currency.

A display price is not enough for a discount at checkout. Stamp `__FB_ATC_UID` on the lines as described in [The discount tag](#the-discount-tag-__fb_atc_uid).

---

## Error responses

The JSON endpoints return `{ ok: false, code, message }`:

| HTTP | `code` | Meaning |
| --- | --- | --- |
| 400 | `INVALID_PARAMS` | `shop` is missing or not a valid `*.myshopify.com` domain (or, on collection-products, `collection_id` or `after` is missing) |
| 404 | `NOT_FOUND` | The headless endpoints are not enabled for this deployment |
| 409 | `APP_NOT_INSTALLED` | The shop has no Storefront token — open Knit once in Shopify admin |
| 502 | `UPSTREAM_ERROR` | The Shopify Storefront API call failed |

`/headless/embed` returns plain-text `400` and `404` bodies for the same missing-parameter and not-enabled cases.

---

## Why the app origin?

Headless clients can't frame `{shop}.myshopify.com`: Shopify sends `Content-Security-Policy: frame-ancestors 'none'` on every storefront response, and merchants can't change it. So the embed page is served from the Knit app origin, which sets no such header.

Because the page lives on the app origin, the widget fetches its data from `/headless/bundles` (same origin, no CORS) rather than the storefront App Proxy.

You may see `https://{shop}.myshopify.com/apps/proxy/bundles` and `/apps/proxy/embed`. Those are HMAC-signed App Proxy endpoints for hosts that **can** frame the storefront origin, and they return the same `Bundle[]` schema. For headless apps and mobile WebViews, use `/headless/*`.

---

## Support

Include your `shop` domain, the endpoint URL, and the full JSON error (or the `[nameless]` console logs from the embed iframe). See [Help](/help).
