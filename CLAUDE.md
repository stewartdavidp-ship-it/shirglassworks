# CLAUDE.md — Shir Glassworks

## Current Idea
**Shir Glassworks Website + Storefront** — Full website redesign and storefront for Shir Glassworks. Phase 1: landing page, about, schedule, admin app with gallery management. Phase 2: shop with product catalog (6 categories, 31 products), analytics tracking. Phase 3: product detail pages with color variants, cart/checkout strategy. Phase 4: Studio Companion — camera-first PWA for production management and craft fair sales. Hybrid photo recognition (TensorFlow.js + Claude Vision), QR code system, GPS-based mode switching (studio/fair/packing), batch counting, new product onboarding via camera, travel inventory for events.

## RULEs — Do not violate these.

- No unbounded Firebase listeners. All reads must use limitToLast(N) or .once('value'). Prevents billing spikes (learned from $17/day incident on another project). _(from: Shir Glassworks Website + Storefront)_
- Admin writes go to shirglassworks/public/gallery (directly public). Firebase rules enforce admin-only writes (auth.uid check) and anonymous read access for public pages. Analytics writes are append-only with field validation. _(from: Shir Glassworks Website + Storefront)_
- generate_claude_md runs on job creation only when stale. Chat must call generate_claude_md(action="get") to check staleness before creating a draft job — if _stale: true, call generate_claude_md(action="push") to regenerate and deliver. If not stale, skip. Rules and constraints change infrequently — regenerating on every job is wasteful. The stale signal is the trigger, not the job creation event itself. _(from: Shir Glassworks Website + Storefront)_
- Shipping address entered at checkout must be validated via Google Places API before Square payment is taken and before CSV is generated for Pirate Ship. Invalid or unvalidated addresses must be corrected before the order can proceed. This prevents failed USPS deliveries on fragile glass shipments. _(from: Shir Glassworks Website + Storefront)_
- After every major build job (any job that adds new Firebase writes, new user inputs, new Firebase paths, new auth flows, new external API calls, or new UI that renders data from Firebase), Claude Code must create a follow-on Security Posture Review maintenance job before marking the build complete. The review must check: (1) Firebase rules for any new paths, (2) hasPermission() guards on all new writes, (3) esc() usage on all new innerHTML injections, (4) input validation on all new user-supplied values, (5) no sensitive data persisted beyond its immediate use. Critical and High findings must be fixed inline. Medium and Low findings must be captured as OPENs. The security job may be batched across multiple small builds but must not be skipped. _(from: Shir Glassworks Website + Storefront)_

## CONSTRAINTs — External realities. Work within these.

- GitHub Pages hosting — static files only, no server-side code. All dynamic behavior via client-side JS + Firebase RTDB. _(from: Shir Glassworks Website + Storefront)_
- Existing Etsy shop (ShirGlassworks) already handles payments, shipping, inventory, and buyer protection. Any checkout strategy must account for this existing sales channel. _(from: Shir Glassworks Website + Storefront)_
- Orders do not currently track variant-level combo keys. Fulfillment operates on stock._default only. Forecasting at the attribute level (e.g., "build more Short Blue cups") requires variant-level fulfillment to be built first. Until then, forecasting can only operate at the product level. _(from: Shir Glassworks Website + Storefront)_
- No historical inventory snapshots exist. Firebase only stores current stock state — no time-series of inventory levels. Cannot answer "what was our stock level 30 days ago" or calculate burn rate from inventory history. Forecasting must derive demand from order history, not inventory deltas. _(from: Shir Glassworks Website + Storefront)_
- Artisan-scale order volume limits statistical significance. This is not a high-volume retail business — a product may sell 2-5 units per month. Forecasting models that rely on large sample sizes (moving averages, regression) may produce noisy results. Simple heuristics may outperform statistical methods at this scale. _(from: Shir Glassworks Website + Storefront)_
- Etsy order history only exists from when sync was first enabled — not full historical data. Direct orders have complete history in Firebase, but Etsy orders before integration are missing. Forecasting must account for incomplete history on the Etsy channel. _(from: Shir Glassworks Website + Storefront)_
- No cost or margin data on products. Product records have selling price but no cost-of-materials, labor time, or margin fields. Forecasting cannot prioritize by profitability — only by demand volume or revenue. Adding cost data would require extending the product or inventory data model. _(from: Shir Glassworks Website + Storefront)_
- RBAC is role-level only — no row-level or attribute-level data filtering. All users with access to a function can see all records that function exposes. Shir Glassworks does not require per-user data scoping for internal roles (Admin, User).

The only data-scoped role is Guest, which is enforced by query design (customers query only their own records by uid), not by the permission matrix. This is not configurable — Guest data isolation is structural.

Do not implement row-level security, record ownership filters, or per-user data visibility controls for Admin or User roles. If this requirement emerges in the future (e.g. multi-location staff who should only see their location's data), it is a new architectural decision at that time. _(from: Shir Glassworks Website + Storefront)_
- Vendor Google integration assumes a single shared Google account used by both Ori and Madeline for Drive and Contacts. Domain accounts (shirglassworks.com) are a future goal but not a current blocker — treat as same-account for all integration design now. _(from: Shir Glassworks Website + Storefront)_

## UI STANDARDS — Follow these in all builds.

### Colours — CSS variables only, no hardcoded values.
**Admin** (`app/index.html`) `:root`: `--amber` (#C4853C, primary CTA), `--amber-light`, `--amber-glow` (focus ring), `--teal` (#2A7C6F, secondary accent), `--teal-deep`, `--teal-light`, `--cream` (#FAF6F0, page bg), `--cream-dark` (#F0E8DB, hover/dividers), `--charcoal` (#1A1A1A, text), `--warm-gray` (#6B6560, secondary text), `--warm-gray-light` (#9B958E, placeholder), `--danger` (#DC3545), `--danger-hover`, `--gold` (#C4853C), `--dark-brown` (#5C3D2E).

**POS** (`pos/index.html`) dark-first `:root`: `--bg` (#1a1a1a), `--card` (#242424), `--border` (#333), `--text` (#e8e0d4), `--muted` (#9a9089), `--gold` (#c4944a), `--green` (#4caf50), `--yellow` (#ffc107), `--red` (#f44336), `--blue` (#42a5f5), `--safe-top`/`--safe-bottom` (iOS safe area insets).

Never write a hardcoded hex, `rgb()`, or colour name in styles. Common fixes: `#fff` → `var(--cream)`/`var(--card)`, `#666` → `var(--warm-gray)`, `#ddd` border → `var(--cream-dark)`/`var(--border)`, `#4CAF50` → `var(--green)`. If no variable exists, add it to `:root` AND the dark mode block first.

### Typography
- **Admin body/UI:** `'DM Sans', sans-serif`. **POS body/UI:** `-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif` (system stack).
- **Display headings (admin only):** `'Cormorant Garamond', serif` — section titles and decorative headings.
- **Type scale:** `0.72rem` (table headers, badges), `0.75rem` (metadata), `0.78rem` (compact buttons), `0.82rem` (dense lists), `0.85rem` (default body), `0.9rem` (prominent body), `1rem` (paragraphs), `1.1rem` (sub-titles), `1.3rem` (modal titles), `2rem` (large display values). Do not invent new sizes.

### Spacing & Border Radius
- **Border radius:** `6px` (default: buttons, inputs, cards), `8px` (softer panels), `4px` (badges, small buttons), `12px` (pills, bottom sheets), `50%` (circles). Do not use arbitrary values like 5px, 7px, 10px.
- **Spacing:** Multiples of 4px (4, 8, 12, 16, 20, 24, 32, 48, 60). Modal body: `20px 24px`. Form group margin-bottom: `16px`.

### Component Patterns
- **Buttons:** Always use `.btn` + modifier (`.btn-primary`, `.btn-secondary`, `.btn-danger`, `.btn-icon`, `.btn-adj`). Never style buttons from scratch with inline styles. Dark mode variants are automatic.
- **Modals:** Use `openModal(html)` / `closeModal()`. Structure: `.modal-header` (h3 + `.modal-close`), `.modal-body`, `.modal-footer` (Cancel secondary + action primary). Never build custom overlay markup.
- **Forms:** Wrap in `.form-group` (16px bottom margin, block label). Required fields: `<span class="field-required">*</span>`. Input border: `var(--cream-dark)` (admin) / `var(--border)` (POS). Border-radius: `6px`. Padding: `8px`.
- **Status badges:** `.status-badge` base (inline-block, 4px radius, 0.72rem, weight 600). `.status-badge.pill` for rounded variant. Role badges: `.role-badge.admin` (purple), `.role-badge.user` (blue), `.role-badge.guest` (amber).
- **Tables:** Use `.data-table`. `th`: 0.72rem uppercase, letter-spacing 0.08em, `var(--warm-gray-light)`.
- **Empty states:** `.empty-state` with `.empty-icon` (contextual emoji, 2.2rem, 0.7 opacity) + `<p>` message. Padding: `60px 20px`.
- **Loading:** `.loading` class with CSS spinner via `::before`. Same sizing as empty state.
- **Toasts:** `showToast(message)` for success, `showToast(message, true)` for errors. Never use `alert()`.

### Z-Index Layers
`98-100` (sticky headers), `200` (dropdowns), `300` (bottom sheets), `1000` (modals), `9999` (toasts), `10000` (critical overlays). Do not invent new layers.

### Shadows
`0 1px 3px rgba(0,0,0,0.08)` (card), `0 2px 8px rgba(0,0,0,0.12)` (hover), `0 3px 10px rgba(0,0,0,0.15)` (modals), `0 8px 30px rgba(0,0,0,0.2)` (overlays), `0 0 0 3px var(--amber-glow)` (focus ring). Use only these values.

### Dark Mode
**Admin:** Class-based (`.dark-mode` on body). All component dark variants defined in stylesheet. **POS:** Always dark — `:root` variables are the dark palette, no override needed. All JS-generated HTML must use CSS variables for every colour/background/border. Inline `style="background:#fff"` will break dark mode and is never acceptable.

### POS-Specific
- Fixed headers/footers must use `padding-top: var(--safe-top)` / `padding-bottom: var(--safe-bottom)` for iPhone notch/home-bar.
- Bottom sheets: `z-index: 300`, `border-radius: 12px 12px 0 0`, backdrop `rgba(0,0,0,0.7)`.
- Touch targets: minimum 44×44px for any tappable element.

### Inline Style Policy
Inline styles are allowed only for dynamic JS-computed values (width%, positioning). Never use inline styles for colours, backgrounds, borders, font sizes, or shadows — use CSS variables or classes. Hardcoded hex in an inline style is a dark mode violation.
