# ARCHITECTURE.md — Shir Glassworks

## Overview

Shir Glassworks is a static website deployed via GitHub Pages with a Firebase Realtime Database backend. The public site (product catalog, gallery, checkout) uses vanilla JS with no framework. The admin app (`/app/index.html`) uses React 18 via CDN.

**Repo:** `stewartdavidp-ship-it/shirglassworks`
**Firebase project:** `word-boxing`
**Firebase paths:** `shirglassworks/public/` (anonymous read), `shirglassworks/admin/` (auth required), `shirglassworks/orders/` (mixed)
**Cloud Functions:** `gameshelf-functions/functions/index.js` (shared with other projects)

## Public Site

Static HTML pages served by GitHub Pages. No build system, no bundler.

| File | Purpose |
|------|---------|
| `index.html` | Homepage / landing |
| `shop.html` | Product catalog grid |
| `product.html` | Single product detail + add-to-cart |
| `checkout.html` | Multi-step checkout flow |
| `checkout.js` | Checkout logic (IIFE, ~1140 lines) |
| `checkout.css` | Checkout-specific styles |
| `about.html`, `gallery.html`, etc. | Content pages |

### Firebase SDK

Uses Firebase compat SDK (v9.22.0) loaded via CDN. All public reads are anonymous — no auth required. Pattern:
```js
firebase.database().ref('shirglassworks/public/...').once('value', snap => { ... });
```

## Checkout Flow

Multi-step checkout implemented in `checkout.js` as an IIFE (`ShirCheckout`).

### Steps

1. **Address** — Customer enters shipping address. Google Places Autocomplete provides type-ahead on the street address field. Soft validation: first attempt warns if not Places-validated, second attempt allows through.

2. **Shipping** — Flat-rate calculation based on product `shippingCategory`. Fetches shipping config from `shirglassworks/public/config/shippingRates` and product shipping data (`weightOz`, `shippingCategory`) for each cart item. Single calculated shipping line displayed (no options to choose from).

3. **Review** — Shows line items, address, shipping, estimated tax, total. Edit links return to prior steps.

4. **Payment** — Calls `shirSubmitOrder` cloud function which creates a Square Payment Link. Customer is redirected to Square's hosted checkout. On completion, Square redirects back to the confirmation page.

5. **Confirmation** — Reads `pendingOrder` from sessionStorage (survives the Square redirect). Firebase listener watches `shirglassworks/orders/{orderId}/status` for transition from `pending_payment` to `placed` (triggered by Square webhook → `shirSquareWebhook` cloud function). On detection, auto-generates and downloads Pirate Ship CSV.

### Shipping Calculation

**Algorithm** (identical on client and server):
- Subtotal ≥ `freeThreshold` → free shipping
- Otherwise: `max(category rate across items) + (additionalItems × surcharge)`
- Categories: `small` ($6), `medium` ($10), `large` ($15), `oversized` ($22)
- Default config hardcoded as fallback if Firebase config missing

**Config path:** `shirglassworks/public/config/shippingRates`
```json
{
  "small": { "rate": 6, "boxL": 6, "boxW": 6, "boxH": 6 },
  "medium": { "rate": 10, "boxL": 10, "boxW": 8, "boxH": 6 },
  "large": { "rate": 15, "boxL": 14, "boxW": 12, "boxH": 8 },
  "oversized": { "rate": 22, "boxL": 20, "boxW": 16, "boxH": 12 },
  "additionalItemSurcharge": 2,
  "freeThreshold": 100,
  "packingBufferOz": 8
}
```

### Google Places Integration

- API key stored at `shirglassworks/public/config/googleMapsApiKey`
- Script loaded lazily at checkout start (not on page load)
- Graceful fallback: if no key or script fails, checkout works without autocomplete
- Restricted to US addresses: `{ types: ['address'], componentRestrictions: { country: 'us' } }`
- `.pac-container` styled to match Shir aesthetic (gold accents, cream background)

### Pirate Ship CSV

Generated client-side from sessionStorage data after payment confirmation.

**Columns:** Name, Company, Address1, Address2, City, State, Zip, Country, Phone, Weight_oz, Length, Width, Height, Description, Order_ID

- Weight = Σ(item.weightOz × qty) + (packingBufferOz × totalItems)
- Box dimensions from highest shippingCategory item's config defaults
- Auto-downloads when Firebase listener detects `placed` status
- Manual "Download Shipping CSV" button as fallback
- Listener auto-detaches after 60s safety timeout

### Test Mode

- Config at `shirglassworks/public/config/testMode` (boolean)
- Set automatically when Square environment = sandbox in admin Settings
- Shows orange banner: "TEST MODE — No real charges will be made"
- Square's own sandbox page shows test card numbers

## Cloud Functions

Located in `gameshelf-functions/functions/index.js` (shared repo).

### `shirSubmitOrder` (~line 2998)

HTTP callable function. Receives order data from checkout, creates Square Payment Link.

1. Validates order items against product catalog in Firebase
2. Calculates subtotal from authoritative product prices (ignores client-sent prices)
3. **Calculates shipping server-side** (authoritative — same algorithm as client)
4. Estimates tax based on state
5. Creates Square Payment Link via API with line items
6. Writes order to `shirglassworks/orders/{orderId}` with status `pending_payment`
7. Returns `{ success, orderId, orderNumber, checkoutUrl }`

### `shirSquareWebhook` (~line 3167)

HTTP endpoint receiving Square webhook notifications.

1. Verifies Square webhook signature
2. On `payment.completed`: updates order status to `placed`, writes payment details
3. Triggers downstream: Firebase listener on confirmation page detects status change → CSV download

## Product Data Model

Path: `shirglassworks/public/products/{pid}`

Key shipping-related fields:
- `weightOz: number` — actual product weight in ounces (default: 16 if unset)
- `shippingCategory: "small" | "medium" | "large" | "oversized"` — drives rate and box dimensions (default: "small" if unset)

These fields are editable in the admin app's product edit form (Production tab).

## Admin App

Single-file React app at `/app/index.html`. Uses React 18 + Tailwind CSS via CDN.

### Settings Tab — Shipping Config

Inputs for all shipping rate/dimension values, save/load to `shirglassworks/public/config/shippingRates`.

### Settings Tab — Google Maps API Key

Text input, saves to `shirglassworks/public/config/googleMapsApiKey`.

### Settings Tab — Square Config

Environment toggle (sandbox/production), Application ID, Location ID per environment. Saves to `shirglassworks/admin/config/squareEnv`. When sandbox selected, also writes `testMode: true` to public config.

**TODO:** Square credentials admin UI with field masking and validation (addendum, not yet built).

## Firebase Paths

| Path | Access | Purpose |
|------|--------|---------|
| `shirglassworks/public/products/` | Anonymous read | Product catalog |
| `shirglassworks/public/gallery/` | Anonymous read | Gallery images |
| `shirglassworks/public/config/shippingRates` | Anonymous read | Shipping calculation config |
| `shirglassworks/public/config/googleMapsApiKey` | Anonymous read | Places API key |
| `shirglassworks/public/config/testMode` | Anonymous read | Sandbox mode flag |
| `shirglassworks/admin/config/squareEnv` | Auth required | Square credentials |
| `shirglassworks/admin/inventory/` | Auth required | Inventory records |
| `shirglassworks/admin/jobs/` | Auth required | Production jobs |
| `shirglassworks/orders/` | Mixed | Order records (write: cloud function, read: status field public for confirmation listener) |

## Key Patterns

- **No framework on public pages.** Vanilla JS, IIFE pattern, Firebase compat SDK.
- **SessionStorage for redirect persistence.** Cart and order data stored in sessionStorage before Square redirect, retrieved on return.
- **Server-side is authoritative.** Client calculates shipping/tax for display; server recalculates from source data before creating payment.
- **Graceful degradation.** Google Places, test mode banner, and shipping config all have fallback defaults if Firebase reads fail.
- **Firebase listeners with safety bounds.** All `.on('value')` listeners have timeout-based detach to prevent unbounded billing.
