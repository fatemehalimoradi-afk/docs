---
title: "Headless Bundles API"
description: "Embed the Fast Bundle 2.0 widget or read bundle data on a headless storefront or mobile app"
---

| Mode | You build |
| --- | --- |
| **Embed** (fastest) | An `<iframe>` and a `postMessage` handler that adds to your cart |
| **Data API** (full control) | Your own UI, price display, and cart logic — including the discount tag |

Both are served from `https://app.fastbundle.co` under `/headless/*`, and every request needs `shop={your-store}.myshopify.com`. Before you start, create an **Active** bundle and open Fast Bundle 2.0 once in Shopify admin; until then, endpoints return `APP_NOT_INSTALLED`.

---

## Endpoint 1 — Bundle data

```
GET https://app.fastbundle.co/headless/bundles?shop={shop}&product_id={product_id}
```

### Query parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | Your `{store}.myshopify.com` domain |
| `product_id` | No | Numeric Shopify product ID (not a GID — a GID stops product sets from matching). Returns only bundles for that product: a product match, collection membership, or its [Bundle as Product](/shopify-integration#bundle-as-product) page. Omit for every bundle |
| `handle` | No | Only the bundle with this handle |
| `type` | No | Omit. Defaults to the current bundle type |
| `language` | No | Language code (`EN`, `DE`, `PT-BR`) for translated titles and `@inContext` pricing. Limits `translations` to that language, its base language, the shop default, and English |
| `country` | No | Country code (`US`, `DE`) for market pricing |

### Example response

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
                "image": null,
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
        "layout": "ANCHORED",
        "jsUrl": "https://…/renderer.js",
        "cssUrl": "https://…/styling.css"
      },
      "assets": []
    }
  ]
}
```

Trimmed; see the [Response schema reference](#response-schema-reference).

- **Not filtered by status.** Skip bundles with `isActive` false, `pdpDisplay.isActive` `"false"`, or (unless you draw a pop-up) `pdpDisplay.layout` `"POPUP"`.
- **Show on → Selected products' pages is not applied**, and Bundle as Product products can appear in other bundles' collections.
- **`pdpDisplay` scalars are strings.** Read `jsUrl`/`cssUrl`, then `pdpDisplay.assets`, then the bundle's `assets`.
- Bundles with a General product set need `product_id`.

### More of a collection's products

A collection's `products` holds the first batch; `nextCursor` is `null` when there are no more.

```
GET https://app.fastbundle.co/headless/collection-products?shop={shop}&collection_id={collection.id}&after={collection.nextCursor}
```

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | As above |
| `collection_id` | **Yes** | `collection.id`, the GID |
| `after` | **Yes** | `collection.nextCursor` |
| `language`, `country` | No | As above |

Returns `{ ok: true, products: Product[], nextCursor }`, up to 50 products. Drop the selector's `excludedVariantIds` yourself (and products left with none).

### Caching

```
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

Changes can take about 6 minutes to reach clients. Errors are not cached.

`/headless/bundles` and `/headless/collection-products` send `Access-Control-Allow-Origin: *` (no credentials), so you can call them from any page.

---

## Endpoint 2 — Embed (iframe)

```
GET https://app.fastbundle.co/headless/embed?shop={shop}&product_id={product_id}
```

Put it in an iframe; you only handle add-to-cart over the [postMessage bridge](#postmessage-bridge).

| Parameter | Required | Description |
| --- | --- | --- |
| `shop` | **Yes** | Your `{store}.myshopify.com` domain |
| `product_id` | Recommended | Numeric product ID; decides which bundles render |
| `language` | No | Page language and translations |
| `country` | No | Market pricing |

`handle`, `type`, and `variant_id` are accepted but have no effect.

The embed skips hidden, pop-up, and unfillable bundles (so it can be blank), ignores **Show on → Selected products' pages**, offers no subscriptions, and always uses the widget's own button.

```html
<iframe
  id="fast-bundle"
  src="https://app.fastbundle.co/headless/embed?shop=your-store.myshopify.com&product_id=7494254788801"
  style="width:100%;border:0;"
  scrolling="no"
></iframe>
```

The page is sent with `Cache-Control: no-store`, can be framed by any origin, and is sandboxed (`Content-Security-Policy: sandbox allow-scripts allow-popups allow-popups-to-escape-sandbox`). A missing or invalid `shop` returns a plain-text `400`.

---

## postMessage bridge

Every message has `source`, `v` (always `1`), and `type`. Ignore messages whose `source` or `v` doesn't match.

### From the iframe to your page

`source: "nameless-bundle"`, `v: 1`

| `type` | Payload | Meaning |
| --- | --- | --- |
| `ready` | — | The widget mounted |
| `resize` | `{ height: number }` | New content height in px. Apply as is — it can shrink |
| `addToCart` | `{ requestId, lineItems }` | The shopper pressed add-to-cart. Add the lines, then reply |

`ready` and `resize` go to both `window.parent` and `window.top`; `addToCart` only to `window.parent`.

### From your page to the iframe

`source: "nameless-bundle-host"`, `v: 1`

| `type` | Payload | Meaning |
| --- | --- | --- |
| `addToCartResult` | `{ requestId, ok, code?, message? }` | Echo the `requestId`. On failure send `ok: false`. `CART_REJECTED` (the default with no `code`) is a refusal; any other code is `CART_HTTP_ERROR`. Your `message` is kept |

With no reply within **10 seconds**, the add fails and the widget shows an error.

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

### Host handler

The sandboxed iframe's messages arrive with `event.origin === "null"`. Identify it by `event.source === iframe.contentWindow` plus `source` and `v`, and reply with target origin `"*"` — a reply aimed at the app origin is dropped and the add times out. Replace the constants with your own values.

```html
<script>
  const STOREFRONT_API = "https://your-store.myshopify.com/api/2026-07/graphql.json";
  const STOREFRONT_TOKEN = "your-storefront-access-token"; // a public Storefront API token
  const IFRAME = document.getElementById("fast-bundle");

  window.addEventListener("message", async (event) => {
    // Only the Fast Bundle 2.0 iframe may talk to this handler (its origin is "null").
    if (event.source !== IFRAME.contentWindow) return;
    const msg = event.data;
    if (!msg || msg.source !== "nameless-bundle" || msg.v !== 1) return;

    if (msg.type === "resize" && Number.isFinite(msg.height) && msg.height >= 0) {
      IFRAME.style.height = msg.height + "px";
      return;
    }

    if (msg.type === "addToCart") {
      if (typeof msg.requestId !== "string" || !validLineItems(msg.lineItems)) return;
      try {
        await addToCart(msg.lineItems);
        reply(msg.requestId, true);
      } catch (err) {
        reply(msg.requestId, false, "CART_REJECTED", err instanceof Error ? err.message : String(err));
      }
    }
  });

  function validLineItems(items) {
    return (
      Array.isArray(items) &&
      items.length > 0 &&
      items.every(
        (item) =>
          Number.isSafeInteger(item.variantId) &&
          Number.isInteger(item.quantity) &&
          item.quantity > 0 &&
          Array.isArray(item.attributes) &&
          item.attributes.every((a) => typeof a.key === "string" && typeof a.value === "string"),
      )
    );
  }

  async function addToCart(lineItems) {
    const lines = lineItems.map((item) => ({
      merchandiseId: "gid://shopify/ProductVariant/" + item.variantId,
      quantity: item.quantity,
      attributes: item.attributes, // keep __FB_ATC_UID, or checkout applies no bundle discount
    }));
    const res = await fetch(STOREFRONT_API, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Shopify-Storefront-Access-Token": STOREFRONT_TOKEN,
      },
      body: JSON.stringify({
        query: `mutation ($cartId: ID!, $lines: [CartLineInput!]!) {
          cartLinesAdd(cartId: $cartId, lines: $lines) { userErrors { message } }
        }`,
        variables: { cartId: await currentCartId(), lines },
      }),
    });
    const json = await res.json();
    const errors = (json.errors || []).concat(json.data?.cartLinesAdd?.userErrors || []);
    if (!res.ok || errors.length > 0) throw new Error(errors[0]?.message || "Cart request failed");
  }

  function reply(requestId, ok, code, message) {
    IFRAME.contentWindow.postMessage(
      { source: "nameless-bundle-host", v: 1, type: "addToCartResult", requestId, ok, code, message },
      "*", // the sandboxed iframe has no origin to target
    );
  }
</script>
```

`currentCartId()` is yours: return the shopper's cart id, creating one with `cartCreate` if needed. For another cart API, replace `addToCart` — see [Embed mode](#embed-mode).

---

## The discount tag (`__FB_ATC_UID`)

Checkout only discounts lines that carry this line-item property; without it, **no bundle discount appears**. The key is `__FB_ATC_UID` (exact spelling, two underscores). The value is pipe-delimited:

```
<bundleId>|<version>|<tierId>|<selectorId>|<atcId>
```

| Segment | Source | Role |
| --- | --- | --- |
| `bundleId` | `bundle.id` | Which bundle to load |
| `version` | `bundle.version` | Revision number; not used to find the bundle |
| `tierId` | `offers[].tiers[].id` | The tier the add was priced at. Checkout tries it and only lower tiers |
| `selectorId` | `selectors[].id` of **this** line | Which selector the line counts for |
| `atcId` | One id per add-to-cart (epoch milliseconds) | Shared by all lines of one add. N copies at once use `<ms>`, `<ms>-2`, … `<ms>-N` |

Use the Data API IDs as is (not GIDs).

### Embed mode

The iframe already puts `__FB_ATC_UID` in each line's `attributes`. Forward them unchanged:

| Your cart API | How to pass it |
| --- | --- |
| Storefront API `cartLinesAdd` / Cart API | `attributes: [{ key: "__FB_ATC_UID", value: "..." }]` |
| Ajax `/cart/add.js` | `properties: { "__FB_ATC_UID": "..." }` |
| Tapcart `cart/add` | Keep the `attributes` array on each line item |

Don't filter underscore-prefixed or "hidden" properties — many cart SDKs drop them by default.

### Data API mode (your own UI)

You set the tag. Checkout's rules:

- **Tag the highest tier that holds.** Tiers of an offer are a ladder in wire order; tagging a lower tier caps the discount.
- **A condition counts a pool.** Units of every selector in `satisfiesFor` count together. Units above `max` are capped, not refused.
- **Clauses are AND, conditions in a clause are OR.** A tier holds when one condition from each of its clauses, and each of its offer's clauses, holds.
- **Rewards** lower the used units. `mode` spreads a money value over the target (`TOTAL`), each selector (`PER_SELECTOR`), or each unit (`PER_UNIT`). A `slice` lowers only part of the units: "buy 3, get 1 free" is 100% off with `{ after: 3, maxUnits: 1 }`. `amount` and `price` are in shop currency.
- **Several offers in one add:** each offer's tier tags the lines it used (a line used by two keeps the first offer's tag); other lines carry the first applied tier.
- **Triggers** (`isTrigger: true`) count units anywhere in the cart and use none. Leave untagged the lines of a selector that only triggers name and no reward targets, unless the add is only such lines. With triggers, tag each offer with the higher of the tier you priced and the tier that would hold if every trigger held.
- **Nothing holds, no trigger:** add the lines untagged.

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

If the tag is present but the discount isn't, see [Troubleshooting](/troubleshooting#discount-missing-at-checkout).

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
  version: number;               // revision number; changes on every merchant edit
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
  assets: Asset[];               // empty when pdpDisplay carries the assets; read pdpDisplay first
  translations?: { v: 1; primary?: string; locales: Record<string, Record<string, string>> };
}

interface PdpDisplay {
  id: string | null;
  isActive: string;              // "false" ⇒ hidden on product pages
  priority: string | null;       // lower = shown first
  layout: string | null;         // "ANCHORED" (inline), "POPUP", or "FULL_WIDTH"
  isPopup?: string;              // only when layout is absent, on displays not yet re-saved
  atcOverride: string;
  injectSelector: string;
  relativePosition: string | null; // "BEFORE" | "AFTER" the matched element
  hideSelectors: string[];
  jsUrl: string | null;
  cssUrl: string | null;
  params: unknown;
  customCss: string | null;
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

---

## Error responses

JSON endpoints return `{ ok: false, code, message }`:

| HTTP | `code` | Meaning |
| --- | --- | --- |
| 400 | `INVALID_PARAMS` | `shop` missing or not a valid `*.myshopify.com` domain (on collection-products, also a missing `collection_id` or `after`) |
| 409 | `APP_NOT_INSTALLED` | No Storefront token — open Fast Bundle 2.0 once in Shopify admin |
| 502 | `UPSTREAM_ERROR` | The Shopify Storefront API call failed ("The shop's catalog could not be read.") |

---

For [support](/help), include your `shop`, the endpoint URL, and the JSON error or `[FastBundle]` console logs.
