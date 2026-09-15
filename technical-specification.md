# gammamarket — Technical Specification

**Status:** Draft for review
**Audience:** Implementers of the `gammamarket` LNbits extension
**Companion document:** `gamma-native-python-extension-proposal.md` (architecture and rationale; this document is the normative build contract)
**Target host:** LNbits ≥ v1.6.x, `dev`-branch extension API
**Primary protocol:** GammaMarkets marketplace protocol, pinned to `market-spec` commit
`5dc79c5db0d41c0bea774debf445cce041192840` (2025-05-10)
**Compatibility protocols:** NIP-99, NIP-15, NIP-17, NIP-44 (v2), NIP-59, NIP-89, NIP-37, NIP-04

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are to be interpreted as
in RFC 2119. Where this specification and the pinned GammaMarkets draft conflict, the
conflict MUST be recorded in §21 (Open Questions) rather than resolved silently.

---

## 1. Scope

This specification defines the complete build contract for the `gammamarket` LNbits
extension:

- canonical domain model and persistence schema;
- Nostr event construction for Gamma/NIP-99 and NIP-15 compatibility output;
- NIP-17/NIP-04 order-message ingestion and response;
- order, inventory, payment, and publication state machines;
- HTTP API surface (admin, public, migration);
- background-task behavior;
- key custody, encryption-at-rest, and security requirements;
- idempotency, reliability, and concurrency requirements;
- test and conformance requirements.

It does **not** define: LNbits core internals, relay implementations, UI visual design,
or the GammaMarkets protocol itself.

---

## 2. Pinned inputs

| Input | Pin |
|---|---|
| GammaMarkets market-spec | commit `5dc79c5` (`main` @ 2025-05-10) |
| NIP-99 | `nostr-protocol/nips` master, 99.md (revision recorded in `tests/fixtures/PINS.md`) |
| NIP-15 | same, 15.md (status: draft/unrecommended — compatibility only) |
| NIP-17 / NIP-44 / NIP-59 | same, 17.md / 44.md (v2 payload) / 59.md |
| NIP-89 / NIP-37 / NIP-04 / NIP-42 | same, 89.md / 37.md / 04.md / 42.md |
| Nostr library | `nostr-sdk` (Python bindings) — version pinned in `pyproject`; no hand-rolled crypto |
| LNbits | verified against `v1.6.2-rc1`, commit `e336fe1` |

All pinned revisions MUST be recorded in `PINS.md` in the extension repository and
surfaced in the merchant settings UI. The GammaMarkets revision MUST be emitted in the
`L`/`l` or equivalent metadata tag the extension defines for itself (see §6.8).

---

## 3. Definitions and identifier scheme

### 3.1 Internal identifiers

- All internal primary keys are **UUIDv4** hex strings generated server-side.
- All protocol-visible `d` tags are independent random identifiers, **not** the internal
  primary key, so that internal IDs never leak into public events.
- `d` tag alphabet: lowercase hex or `[a-z0-9-]` slugs; length 8–64 chars.
- Imported NIP-15 products keep their existing NIP-15 product ID as the `d` tag unless
  invalid per §6.6.

### 3.2 Protocol addresses

```text
product:     30402:<merchant_pubkey>:<product_d>      (Gamma)
             30018:<merchant_pubkey>:<product_d>      (NIP-15 compat, same d)
collection:  30405:<merchant_pubkey>:<collection_d>
shipping:    30406:<merchant_pubkey>:<shipping_d>
stall:       30017:<merchant_pubkey>:<stall_d>        (one per catalog)
app rec:     31989:<merchant_pubkey>:<handler_d>
app info:    31990:<merchant_pubkey>:<handler_d>
```

### 3.3 Order identifiers

- `order.id` — internal UUIDv4, never exposed in Nostr traffic or public URLs.
- `order.external_id` — the buyer-supplied `order` tag value for protocol orders; the
  extension-generated UUID for web orders. Uniqueness enforced per
  `(merchant_id, protocol, buyer_pubkey, external_id)` — see §14.
- `order.public_token` — 256-bit CSPRNG token, base64url-encoded, required for all
  public order-status lookups. Stored as a SHA-256 hash, not in cleartext.

### 3.4 Money representation

- Product and shipping prices are stored as `(amount_minor: INTEGER, currency: TEXT,
  currency_decimals: INTEGER)`. ISO 4217 decimals; `decimals=0` for sat-denominated
  items, 2 for USD/EUR, etc.
- All computed order totals are stored in **satoshis** (`*_sat` columns, INTEGER ≥ 0).
- Fiat→sat conversion uses the LNbits exchange-rate service at invoice-creation time;
  the applied rate MUST be persisted on the order (`fx_rate_msat_per_unit` +
  `fx_source` + `fx_timestamp`) for audit.
- Rounding rule: each line item converts and rounds **up** to the next sat; the order
  total is the sum of rounded lines plus rounded shipping. Rounding up avoids the
  merchant absorbing sub-sat losses.

---

## 4. Persistence schema

All tables live in the extension's namespaced database. Types are given in portable
terms (`TEXT`, `INTEGER`, `BIGINT`, `BOOLEAN`, `TIMESTAMP`, `BLOB`) mapped to
SQLite/Postgres by the migration layer. `TIMESTAMP` values are UTC epoch seconds
unless noted.

### 4.1 `merchants`

| column | type | notes |
|---|---|---|
| id | TEXT PK | internal UUID |
| user_id | TEXT NOT NULL | LNbits user id; UNIQUE |
| pubkey | TEXT NOT NULL UNIQUE | 64-hex schnorr pubkey |
| key_ref | TEXT NOT NULL | key-store handle, not key material |
| display_name | TEXT | |
| profile_json | TEXT | kind-0 profile fields |
| payment_preference | TEXT NOT NULL DEFAULT 'manual' | `manual` only in v1 |
| recommended_app_d | TEXT | `d` for the merchant's 31989/31990 pair |
| wallet_id | TEXT NOT NULL | LNbits wallet used for invoices |
| active | BOOLEAN NOT NULL DEFAULT true | |
| created_at / updated_at | TIMESTAMP | |

- `wallet_id` MUST belong to `user_id`; enforced on write and re-validated before each
  invoice creation (wallets can be transferred/deleted).

### 4.2 `catalogs`

`id` PK, `merchant_id` FK, `name`, `description`, `default_currency`,
`default_location`, `nip15_stall_d` (NULL until published), `publish_gamma` BOOL,
`publish_nip15` BOOL, `created_at`, `updated_at`.

### 4.3 `products`

| column | type | notes |
|---|---|---|
| id | TEXT PK | |
| merchant_id | TEXT NOT NULL FK | |
| catalog_id | TEXT NOT NULL FK | |
| d_tag | TEXT NOT NULL | UNIQUE(merchant_id, d_tag) |
| parent_product_id | TEXT NULL FK→products.id | variations only |
| product_type | TEXT NOT NULL | `simple`/`variable`/`variation` |
| format | TEXT NOT NULL | `digital`/`physical` |
| title / summary / description_md | TEXT | |
| amount_minor / currency / currency_decimals | | §3.4 |
| recurring_frequency | TEXT NULL | ISO 8601 duration designator (D/W/M/Y) |
| visibility | TEXT NOT NULL | `hidden`/`on-sale`/`pre-order` |
| nip99_status | TEXT NOT NULL | `active`/`sold` |
| draft | BOOLEAN NOT NULL DEFAULT false | NIP-37 draft; never published as 30402 |
| stock_on_hand | INTEGER NULL | NULL = unlimited |
| stock_reserved | INTEGER NOT NULL DEFAULT 0 | |
| location / geohash | TEXT NULL | |
| weight_value / weight_unit | | ISO 80000-1 |
| dim_l / dim_w / dim_h / dim_unit | | |
| published_at | TIMESTAMP NULL | first 30402 publication |
| revision | INTEGER NOT NULL DEFAULT 0 | incremented on every mutation; feeds outbox |
| created_at / updated_at | | |

Invariants (enforced by CHECK constraints where the DB allows, plus service-layer
assertion):

```text
stock_on_hand IS NULL OR stock_on_hand >= 0
stock_reserved >= 0
stock_on_hand IS NULL OR stock_reserved <= stock_on_hand
product_type = 'variation' ⟺ parent_product_id IS NOT NULL
parent chain depth = 1 (a variation cannot be a parent)
```

### 4.4 Product detail tables

- `product_images` (product_id FK, url, dimensions, sort_order)
- `product_specs` (product_id FK, key, value)
- `product_categories` (product_id FK, category)
- `product_collections` (product_id FK, collection_id FK, UNIQUE pair)
- `product_shipping` (product_id FK, ref_kind `30406|30405`, ref_id FK,
  `extra_cost_minor` NULL) — explicit inheritance; no implicit cascade.

### 4.5 `collections`

`id` PK, `merchant_id` FK, `d_tag`, `title`, `description`, `image`, `location`,
`geohash`, `revision`, timestamps. UNIQUE(merchant_id, d_tag).

`collection_shipping` (collection_id FK, shipping_option_id FK) holds the options a
collection advertises; membership rows live in `product_collections`.

### 4.6 `shipping_options`

`id` PK, `merchant_id` FK, `d_tag` (UNIQUE per merchant), `title`, `description`,
`base_price_minor`, `currency`, `service` (`standard|express|overnight|pickup`),
`carrier`, `countries` (JSON array of ISO 3166-1 alpha-2), `regions` (JSON array of
ISO 3166-2), `duration_min`, `duration_max`, `duration_unit` (`H|D|W`),
`weight_min/_max` + unit, `dim_min/_max` (3 components + unit), `location`, `geohash`,
`active`, `revision`, timestamps.

### 4.7 `orders`

| column | notes |
|---|---|
| id PK | internal UUID |
| merchant_id FK | |
| buyer_pubkey | NULL for web orders |
| protocol | `gamma`/`nip15`/`web` |
| external_id | §3.3 |
| source_event_id | outer 1059 or kind-4 event id that created the order; NULL for web |
| currency / fx_* | snapshot fields |
| subtotal_sat / shipping_sat / total_sat | merchant-computed, never buyer-supplied |
| buyer_amount_sat | the `amount` tag the buyer claimed, for audit/dispute display |
| state | §7.1 enum |
| shipping_state | §7.2 enum |
| contact_json | email/phone/etc., encrypted-at-rest fields §12.4 |
| address_enc | encrypted shipping address blob (§12.4); NULL for digital |
| shipping_option_id FK | NULL for digital/pickup-na |
| payment_hash | UNIQUE; set when invoice created |
| invoice_expiry | |
| public_token_hash | §3.3 |
| payment_exception | BOOLEAN — late/mismatched payment flag |
| created_at / updated_at | |

`order_items` (order_id FK, product_id FK, product_d snapshot, title snapshot,
quantity, unit_price_minor, currency snapshot, line_total_sat).
`order_events` (order_id FK, from_state, to_state, actor `merchant|buyer|system|nostr`,
detail_json, created_at) — append-only audit log; every state transition writes one row
in the same transaction.

### 4.8 `payments`

`id` PK, `order_id` FK, `payment_hash` UNIQUE, `checking_id`, `bolt11`,
`amount_sat`, `status` (`pending|settled|expired|failed`), `settled_at`,
`created_at`. Payments are the join between LNbits callbacks and orders.

### 4.9 `inbox_events`

`id` PK, `outer_event_id` UNIQUE, `rumor_id` UNIQUE-where-present, `merchant_id` FK,
`received_at`, `kind`, `author_pubkey`, `processed_state`
(`received|validated|rejected|processed|quarantined`), `reject_reason`,
`raw_json` (bounded — see §15), `processed_at`.

The inbox is a durable queue: events are persisted **before** processing so restarts
cannot lose them.

### 4.10 `outbox_events`

`id` PK, `merchant_id` FK, `aggregate_type` (`product|collection|shipping|merchant|order_msg`),
`aggregate_id`, `aggregate_revision`, `event_kind`, `event_address` NULL,
`payload_json` (unsigned event template or order-message descriptor — **never** a
long-lived signed event), `state`
(`pending|claimed|publishing|partially_published|published|superseded|failed`),
`attempts`, `next_attempt_at`, `claimed_by`/`claimed_at` (worker lease),
`created_at`, `updated_at`.

`relay_publications`: `id` PK, `outbox_event_id` FK, `relay_url`,
`result` (`accepted|rejected|timeout`), `message` (relay OK message, truncated to 512
chars), `attempted_at`.

### 4.11 `relay_configs`

`id` PK, `merchant_id` FK (NULL = server-wide default), `relay_url`, `direction`
(`public|inbox|both`), `enabled`, timestamps. The kind-10050 discovered inbox relays of
*buyers* are cached in `peer_relays` (`pubkey`, `relay_url`, `fetched_at`, `expires_at`).

### 4.12 `settings` and `migration_jobs`

`settings`: key/value per merchant (e.g., `spec_revision`, feature toggles).
`migration_jobs`: `id`, `merchant_id`, `strategy`, `state`
(`preview|validated|executing|awaiting_cutover|done|aborted`), `manifest_json`,
`created_at`, `updated_at`.

### 4.13 Indexes (minimum)

```text
products(merchant_id, catalog_id)        orders(merchant_id, state)
orders(payment_hash) UNIQUE              inbox_events(outer_event_id) UNIQUE
outbox_events(state, next_attempt_at)    outbox_events(aggregate_type, aggregate_id, aggregate_revision)
payments(payment_hash) UNIQUE            peer_relays(pubkey)
orders(merchant_id, protocol, buyer_pubkey, external_id) UNIQUE
```

---

## 5. HTTP API surface

Base path: `/gammamarket/api/v1`. Admin routes require LNbits user auth and are scoped
to the caller's merchant(s). Public routes are unauthenticated and rate-limited (§15).

### 5.1 Admin — merchant

```text
POST   /merchants                          create merchant (generates or imports key)
GET    /merchants/current                  current user's merchant + relay health summary
PATCH  /merchants/{id}                     profile, payment_preference, wallet_id, toggles
POST   /merchants/{id}/keys/import         body: {nsec} — over TLS only; see §12
POST   /merchants/{id}/publish             enqueue republication of all aggregates
GET    /merchants/{id}/relay-health        per-relay connection/ACK summary
DELETE /merchants/{id}                     deactivate; publishes kind-5 + tombstones
```

### 5.2 Admin — catalog

```text
GET|POST            /catalogs
GET|PATCH|DELETE    /catalogs/{id}
GET|POST            /products
GET|PATCH|DELETE    /products/{id}
POST                /products/{id}/images
GET|POST|PATCH|DELETE /collections[/{id}]
GET|POST|PATCH|DELETE /shipping[/{id}]
GET                 /products/{id}/events      dry-run: rendered unsigned events per protocol
DELETE on a product = soft-delete → draft/hidden + kind-5 or inactive policy per §6.7.
```

### 5.3 Admin — orders

```text
GET   /orders?state=&protocol=
GET   /orders/{id}                     full detail incl. decrypted address for owner
POST  /orders/{id}/status              body: {to_state} — must be a legal transition §7.1
POST  /orders/{id}/shipping            body: {shipping_state, tracking?, carrier?, eta?}
POST  /orders/{id}/cancel              merchant-initiated cancel with reason
POST  /orders/{id}/resolve-exception   resolves payment_exception (refund|accept|reject)
GET   /orders/{id}/events              audit log
```

### 5.4 Public — catalog and checkout

```text
GET   /public/merchants/{pubkey}                    profile + preferences (no internals)
GET   /public/products/{merchant_pubkey}/{d_tag}    rendered listing + availability
GET   /public/collections/{merchant_pubkey}/{d_tag}
GET   /public/shipping/{merchant_pubkey}/{d_tag}
POST  /public/checkout
GET   /public/orders/{public_token}                 status, invoice, state — token-gated
GET   /p/{merchant_pubkey}/{d_tag}                  buyer-facing HTML product page (ui_route)
```

Buyer browsers poll `GET /public/orders/{public_token}` (recommended interval
5s) until `state` reaches `confirmed` or a terminal state. Because the token
appears in the request path, deployments SHOULD exclude it from access logs;
the token MAY alternatively be supplied via `X-Order-Token` header on
`GET /public/order-status`.

`POST /public/checkout` request:

```jsonc
{
  "merchant_pubkey": "<hex>",
  "items": [{"d_tag": "<product d>", "quantity": 1}],
  "shipping_option_d": "<d or null>",
  "address": {...},              // required iff any item is physical
  "email": "...", "phone": "..." // optional contact
}
```

Response `201`:

```jsonc
{
  "public_token": "<256-bit token>",
  "order": {"state": "awaiting_payment", "total_sat": 12345,
            "bolt11": "lnbc…", "payment_hash": "<hex>", "expires_at": 1730000000}
}
```

The public order endpoint MUST return only: state, shipping_state, total_sat, bolt11,
payment status, item summaries, and expiry. It MUST NOT return merchant internals,
internal IDs, buyer address echoes, or other orders' data.

### 5.5 Migration

```text
POST  /import/nostrmarket/preview     body: {strategy: api|json|nostr, payload?}
POST  /import/nostrmarket/execute     body: {job_id, confirmations:{…}}
GET   /import/{job_id}
POST  /import/{job_id}/cutover        final step; performs single-writer switch §13
```

### 5.6 Error model

All errors return RFC 9457 problem details:
`{"type": "urn:gammamarket:<code>", "title": …, "status": …, "detail": …}`.
Defined codes include: `insufficient-stock`, `invalid-transition`,
`duplicate-order`, `wallet-mismatch`, `product-inactive`, `rate-limited`,
`invalid-shipping-destination`, `order-expired`, `unauthorized`.

---

## 6. Nostr event construction

All events are built **unsigned** from current domain state and signed immediately
before publication (§10.3). `created_at` is set at signing time, never at build time.

### 6.1 Product — kind 30402

```jsonc
{
  "kind": 30402,
  "content": "<description_md>",
  "tags": [
    ["d", "<product.d_tag>"],
    ["title", "<title>"],
    ["price", "<decimal amount>", "<CURRENCY>", "<freq?>"],   // required
    ["type", "<simple|variable|variation>", "<digital|physical>"],
    ["visibility", "<hidden|on-sale|pre-order>"],
    ["stock", "<max(0, on_hand - reserved)>"]                  // finite stock only
    ["summary", "<summary>"],
    ["image", "<url>", "<WxH or \"\">", "<sort?>"]*,           // per product_images
    ["spec", "<key>", "<value>"]*,
    ["weight", "<v>", "<unit>"]?,
    ["dim", "<l>x<w>x<h>", "<unit>"]?,
    ["location", "<string>"], ["g", "<geohash>"],
    ["t", "<category>"]*,
    ["a", "30402:<pubkey>:<parent d>"]?,                       // variations only, exactly one
    ["a", "30405:<pubkey>:<collection d>"]*,                   // explicit membership
    ["shipping_option", "30406:<pubkey>:<d>", "<extra-cost?>"]*,
    ["shipping_option", "30405:<pubkey>:<d>", "<extra-cost?>"]*// collection shipping refs
  ]
}
```

Rules:

- A `variation` product MUST emit exactly one `a` tag to its `variable` parent and MUST
  NOT itself be referenced as a parent.
- `stock` tag omitted when `stock_on_hand IS NULL` (unlimited).
- Products with `draft=true` MUST NOT produce a 30402; see §6.7 for draft handling.
- `visibility=hidden` products still publish (per spec, visibility is a display hint)
  but are excluded from the web catalog. A `hidden` product MUST still reject new
  public-checkout orders.
- When `available` reaches 0 the service SHOULD set `nip99_status=sold` (and restore
  `active` when stock is replenished) so NIP-99 clients reflect sellability; the
  `stock` tag alone is not a reliable sellability signal across clients.
- `sold` (`nip99_status`) maps to the NIP-99 `status` tag per the pinned NIP-99
  revision; verify tag name at build time against `PINS.md`.

### 6.2 Collection — kind 30405

Required tags: `d`, `title`, one `a` per member product (`30402:<pubkey>:<d>`).
Optional: `image`, `summary` (first 280 chars of description), `location`, `g`,
`shipping_option` → `30406` refs. Content = description markdown.

### 6.3 Shipping option — kind 30406

Required tags: `d`, `title`, `price` `[base_cost, currency]`, `country` (ISO 3166-1
alpha-2 list), `service`. Optional per spec: `region`, `duration` `[min,max,H|D|W]`,
`carrier`, `location`, `g`, `weight-min/max`, `dim-min/max`, `price-weight`,
`price-volume`, `price-distance`.

Validation before publish: `service=pickup` requires `location` or `g`; constraint
min ≤ max; countries non-empty.

### 6.4 Merchant profile — kind 0

Standard profile content plus `["payment_preference", "manual"]` tag. `ecash`/`lud16`
MUST NOT be advertised in v1 (extension only honors `manual`).

### 6.5 Application handler pair — kinds 31989/31990

- `31990` (handler information): `d` = `merchant.recommended_app_d`; content describes
  the extension's public checkout; MUST include a `["web", "<checkout URL template>",
  "nevent"/"naddr"]` handler entry per NIP-89 so Nostr buyers can be redirected to the
  web checkout for `manual` payment flow.
- `31989` (recommendation): `d` = same; `a` tag pointing at
  `31990:<merchant_pubkey>:<recommended_app_d>`.

### 6.6 NIP-15 compatibility events

**Stall 30017** — one per catalog; `d` = `catalog.nip15_stall_d`; content JSON:

```jsonc
{"id": "<stall_d>", "name": "<catalog.name>", "description": "<…>",
 "currency": "<default_currency>",
 "shipping": [{"id": "<opt_d>", "name": "<title>", "base": <minor>, "countries": […], "regions": […]}]}
```

**Product 30018** — content JSON per NIP-15: `{id: <d_tag>, stall_id: <stall_d>,
name, description, images[], currency, price, quantity: available, specs{},
shipping: [{id, cost}] }`. Lossy rules (must be surfaced in UI preview):

- product in multiple collections → belongs to exactly one stall (its catalog's);
- variations → flattened: each variation becomes an independent 30018 with suffix
  `-<variation_d>` on its NIP-15 `id`, `specs` carrying the option values;
- `extra-cost` shipping → NIP-15 `shipping[].cost` added to the zone base;
- `hidden`/`pre-order` → `quantity: 0` + UI flag rather than deletion, to avoid
  address churn.

### 6.7 Deletion and drafts

- Product/collection/shipping removal publishes `kind:5` with `["a", "<address>"]`
  and `["k", "<kind>"]` tags, then clears `ProtocolAddress.latest_event_id`.
- Deleting a `shipping_option` or `collection` still referenced by products MUST be
  either rejected with a reference report or executed as an explicit strip-and-
  republish (merchant chooses); dangling `30406`/`30405` references in published
  products are a defect.
- Drafts follow NIP-37: kind `31234` draft events MAY be published for cross-device
  merchant drafts; draft content is NIP-44-encrypted to self. Drafts MUST NOT be
  referenced by 30405 `a` tags.

### 6.8 Self-describing metadata

Every published event MUST carry `["L", "gammamarket"]` and
`["l", "<pinned spec revision short-sha>", "gammamarket"]` (NIP-32 labels) so future
spec revisions are detectable in the wild. Exact label namespace finalized in Phase 0.

### 6.9 Order messages (rumors)

All Gamma order traffic is NIP-17 gift-wrapped (kind 1059 → seal 13 → rumor).
Rumor `created_at` = real time; outer/gift timestamps randomized per NIP-59.

**kind 16, type 1 — order creation (buyer→merchant)** — required tags `p`
(merchant), `subject`, `type=1`, `order`, `amount` (sats), `item` ×n
(`["item","30402:<pk>:<d>","<qty>"]`); optional `shipping`, `address`,
`email`, `phone`. Inbound `subject` is required by the market-spec but the
extension MUST tolerate its absence (treat as empty) for lenient interop.

**kind 16, type 2 — payment request (merchant→buyer)** — `p` (buyer),
`subject="order-payment"`, `type=2`, `order`, `amount` = merchant-computed
total_sat, `["payment","lightning","<bolt11>"]`, `expiration`.

**kind 16, type 3 — status (both directions)** — `subject="order-info"`, `type=3`,
`order`, `status` ∈ {pending, confirmed, processing, completed, cancelled}.

**kind 16, type 4 — shipping (merchant→buyer)** — `subject="shipping-info"`,
`type=4`, `order`, `status` ∈ {processing, shipped, delivered, exception};
optional `tracking`, `carrier`, `eta`.

**kind 17 — receipt (buyer→merchant)** — `order`,
`["payment","lightning","<bolt11>","<preimage>"]`, `amount`. Treated as a **hint
only** — never a settlement trigger (§8.3).

**kind 14 — general DM** — stored, surfaced in merchant UI; optional `subject` =
order id.

### 6.10 NIP-15 message compatibility (NIP-04 DMs)

| NIP-15 type | direction | content JSON |
|---|---|---|
| 0 order | buyer→merchant | `{id, name, address, message, contact:{nostr?,email?,phone?}, items:[{product_id, quantity}]}` |
| 1 payment req | merchant→buyer | `{id, message, payment_options:[{type:"ln", link:"lightning:<bolt11>"}]}` |
| 2 status | both | `{id, message, paid: bool, shipped: bool}` |

NIP-04 (kind 4) is deprecated-insecure: accepted for compatibility, but the merchant
UI MUST label the channel "legacy / metadata-exposed".

---

## 7. State machines

### 7.1 Order state

```text
received → rejected | awaiting_payment
awaiting_payment → confirmed | expired | cancelled
confirmed → processing | cancelled          (cancel after confirm = exceptional, needs reason)
processing → completed | cancelled
completed, rejected, expired, cancelled → terminal
```

- Legal transitions are enforced by a single `transition_order()` function used by
  every entry path (API, inbox, tasks). No code path may write `orders.state`
  directly.
- Buyer-initiated cancel (kind 16 type 3 `cancelled`) is honored only from
  `received`/`awaiting_payment`; after `confirmed` it is recorded but requires
  merchant action.
- `payment_exception=true` is orthogonal to state: set when settlement arrives for a
  terminal or mismatched order (§8.4).

Gamma projection (for outbound type-3):
`received|awaiting_payment→pending`, `confirmed→confirmed`, `processing→processing`,
`completed→completed`, `rejected|expired|cancelled→cancelled` + reason in content.

NIP-15 projection: `paid` = state ≥ confirmed (not terminal-cancelled); `shipped` =
shipping_state ∈ {shipped, delivered}.

### 7.2 Shipping state

`not_required → pending → processing → shipped → delivered`; `exception` reachable
from `processing|shipped`. Digital orders start and stay `not_required`.

### 7.3 Reservation state

`held → consumed | released | expired`. Only `held` reservations count in
`stock_reserved`. `consumed` is applied exactly once inside the settlement
transaction.

### 7.4 Outbox state

`pending → claimed → publishing → published | partially_published | failed`;
`pending|claimed|publishing` → `superseded` when a newer `aggregate_revision` exists
for the same `(aggregate_type, aggregate_id, event_kind)`. Order-message outbox rows
(`aggregate_type=order_msg`) MUST NOT be superseded.

### 7.5 Inbox state

`received → validated → processed`; `→ rejected` (bad signature/shape, terminal);
`→ quarantined` (oversize/malformed/undecryptable, retained for inspection).

---

## 8. Core algorithms

### 8.1 Order intake (all protocols)

1. Parse and bound input (§15 limits).
2. Resolve merchant; reject if `active=false`.
3. Resolve each `item` reference to a canonical product owned by that merchant;
   reject cross-merchant references.
4. Validate: products `on-sale` or `pre-order` (both purchasable; `hidden` is
   not), not draft, `nip99_status=active`; quantities ≥ 1; a `variable` parent is
   never directly purchasable — orders must name a `variation` or `simple`
   product; a `variation` is purchasable only while its parent is `variable`
   and itself purchasable.
5. Recalculate price from current domain state (§3.4). Buyer `amount` tag is stored
   as `buyer_amount_sat` and never trusted; if it differs from `total_sat`, proceed
   with merchant total and note the difference in the payment-request content.
6. Validate shipping option: owned by merchant, active, serves the destination
   country/region, product weight/dims within constraints; compute shipping cost
   (base + extra-cost + weight/volume surcharges per the option's pricing tags).
7. Insert `orders` row (`received`), `order_items`, `order_events` — one transaction.
   The UNIQUE(merchant_id, protocol, buyer_pubkey, external_id) constraint makes
   retries idempotent; on conflict, return the existing order.

### 8.2 Reservation + invoice (transition `received → awaiting_payment`)

One DB transaction:

```sql
-- conditional stock claim per item (finite stock only)
UPDATE products SET stock_reserved = stock_reserved + :qty
WHERE id = :pid AND (stock_on_hand IS NULL OR stock_on_hand - stock_reserved >= :qty)
-- if rowcount != 1 → reject whole order, release prior claims in same txn
```

Then insert `inventory_reservations` (`held`, `expires_at = now + RESERVATION_TTL`),
transition order, commit. Only after commit: create the LNbits invoice bound to
`order.id` + `payment_hash`, persist `payments` row, enqueue the type-2 payment
request (protocol orders) or return bolt11 (web).

- `RESERVATION_TTL` default 15 min; invoice expiry MUST equal reservation expiry so
  stock cannot be held by an expired invoice.
- If invoice creation fails after commit → transition to `rejected` and release
  reservations in a compensating transaction.

### 8.3 Settlement (LNbits invoice-paid event)

Trigger: LNbits `invoice-paid` internal event filtered by `payment_hash ∈ payments`.

One transaction:

1. `SELECT … FOR UPDATE` (or dialect equivalent) the payment row; if already
   `settled` → no-op (idempotent).
2. Mark payment settled; verify `amount_sat` matches invoice amount exactly
   (mismatch → `payment_exception`, do not confirm).
3. Consume each `held` reservation → `consumed`; `stock_on_hand -= qty`,
   `stock_reserved -= qty`.
4. Transition order → `confirmed`; write `order_events` row.
5. Enqueue outbox intents: type-3 `confirmed` message (protocol orders), stock/state
   republication for affected products (new `aggregate_revision`).

If the order is already terminal (`expired`/`cancelled`): mark payment settled, set
`payment_exception=true`, DO NOT change state or consume reservations — merchant
resolves via `/resolve-exception` (accept → confirm + consume; reject → refund
flow out of scope for v1 automation, surfaced as manual task).

### 8.4 Late payment

If invoice expires unpaid → expiry worker releases reservations (`expired`) and
transitions order `expired`, publishing type-3 `cancelled`. A subsequent settlement
follows §8.3's terminal-order branch. Under no code path may a late payment
auto-reopen the order.

### 8.5 NIP-17 ingest pipeline (inbox worker)

For each `inbox_events` row in `received`:

1. Bound-check raw event size and tag count BEFORE any decode (§15).
2. Verify outer 1059 Schnorr signature + id; dedupe `outer_event_id`.
3. NIP-44-decrypt content with `merchant` conversation key → seal (kind 13).
4. Verify seal signature + id; check `seal.pubkey` — reject if it doesn't verify.
5. NIP-44-decrypt seal content → rumor (kind 14|16|17).
6. **Identity check:** `rumor.pubkey == seal.pubkey`; else reject. (This binds the
   plaintext author to the seal signer; the outer gift-wrap key is ephemeral and
   MUST NOT be used for identity.)
7. Validate rumor per §6.9 required tags; enforce `p` tag == merchant pubkey.
8. Dedupe `rumor_id`; insert/match order via §8.1.
9. Mark `processed`; on any cryptographic failure → `quarantined` + reason, never
   retried.

Buyer→merchant cancel (type 3 `cancelled`): apply only via §7.1 legality, keyed to
`order` tag + `rumor.pubkey == order.buyer_pubkey`.

### 8.6 Outbox publish algorithm

Worker loop:

1. Atomically claim up to N rows: `state=pending AND next_attempt_at<=now`, setting
   `claimed_by=<worker id>`, `claimed_at=now`, `state=claimed`. (Single UPDATE …
   WHERE id IN (SELECT …) — portable across SQLite/Postgres.)
2. For each claimed row: if a newer `aggregate_revision` exists for a supersedable
   aggregate → mark `superseded`, continue.
3. Build unsigned event(s) from **current** domain state (not stale payload) for
   catalog aggregates; use stored payload for `order_msg` rows.
4. Sign via key store (§12) with fresh `created_at`.
5. Resolve target relays: `public` set for catalog events; recipient kind-10050 set
   for gift wraps (fetched via `peer_relays` cache, refreshed on failure).
6. Publish; record one `relay_publications` row per relay.
7. Outcome: all required relays OK → `published`; ≥1 OK → `partially_published`;
   none → `attempts++`, `next_attempt_at = now + backoff(attempts)`, back to
   `pending` (or `failed` after `MAX_ATTEMPTS`, surfaced in UI).

Backoff: `min(2^attempts * 5s, 30min) + jitter(0–5s)`. `MAX_ATTEMPTS` = 20.
Stuck `claimed` rows (lease older than 10 min) are reclaimable by any worker —
handles worker crash mid-claim.

### 8.7 Reconciliation (startup + periodic 60s)

- For each `payments.status=pending`: query LNbits payment status; if settled → run
  §8.3; if expired → run expiry path.
- `outbox` rows in `claimed` with stale lease → back to `pending`.
- `inbox` rows `received` older than grace period → reprocess.
- `reservations` expired with order still `awaiting_payment` → expire (§8.4).
- Orders in `awaiting_payment` whose `payments` row exists but whose payment-
  request outbox row was never enqueued (crash between invoice creation and
  enqueue) → re-enqueue the type-2 request / resurface bolt11 on the order.

This covers the crash window between Lightning settlement and callback delivery.

---

## 9. Nostr transport

### 9.1 Interface

```python
class NostrTransport(Protocol):
    async def publish(self, event: SignedEvent, relay_urls: list[str]) -> list[RelayResult]
    async def subscribe(self, name: str, filters: list[Filter],
                        on_event: Callable[[Event], Awaitable[None]]) -> None
    async def close_subscription(self, name: str) -> None
    async def health(self) -> dict[str, RelayHealth]
```

Primary implementation: `nostr-sdk` client with per-purpose connection pools
(public pool vs per-recipient inbox pool). Optional `nostrclient` adapter MAY handle
public-catalog fan-out behind the same interface; it MUST NOT be used for NIP-17
traffic until it supports per-publication relay targeting.

### 9.2 Subscriptions

Per active merchant, on the merchant's `inbox` relay set:

```text
{ "kinds": [1059], "#p": [merchant_pubkey], "since": <cursor> }
{ "kinds": [4],    "#p": [merchant_pubkey], "since": <cursor> }   # NIP-15 compat
```

**Cursor rule:** NIP-59 randomizes the outer 1059 `created_at` (up to ~2 days of
skew), so `since` MUST NOT be derived from last-processed event timestamps.
`since` = `(last subscription open time − 3 days)`; rely entirely on
`outer_event_id` dedup for replay suppression. The persisted cursor tracks
subscription session time, not event time.

### 9.3 Peer inbox relays (kind 10050)

Before sending any gift wrap: lookup `10050:<buyer_pubkey>` from the merchant's
public pool; cache in `peer_relays` (TTL 24h). If absent: fallback = merchant's
inbox set + public set (documented degradation, flagged in order_events). Gift wraps
MUST be published ONLY to the resolved recipient set — never to the full public pool.

### 9.4 NIP-42

If a relay requires AUTH, authenticate with the merchant key via key store signing;
auth challenge strings are relay-provided — treat as untrusted input.

---

## 10. Background tasks

All tasks registered as LNbits permanent tasks with unique names
(`gammamarket.<task>`). Under multi-worker deployment, each loop MUST take a DB
lease (`settings` row `task_lease:<name>` = worker id + expiry) so exactly one
worker runs it (§14.4).

| task | cadence | purpose |
|---|---|---|
| `relay_manager` | event-driven + 30s health tick | connect/reconnect pools, restore subs, exponential backoff w/ jitter |
| `inbox_processor` | on-event + drain loop | §8.5 pipeline over `inbox_events` |
| `outbox_publisher` | 5s poll | §8.6 |
| `invoice_listener` | LNbits payment event | §8.3 settlement trigger |
| `reservation_expiry` | 30s | release expired `held` reservations |
| `reconciliation` | startup + 60s | §8.7 |

Every task: bounded queue depth, structured logs (IDs and states only), panic-safe —
a task exception must not kill the extension; failed units are marked, not dropped.

---

## 11. Key custody and cryptography

### 11.1 `MerchantKeyStore` interface

```python
generate(merchant_id) -> pubkey
import_key(merchant_id, nsec_bech32) -> pubkey
public_key(merchant_id) -> pubkey
sign_event(merchant_id, event_dict) -> signed_event       # hashes + schnorr signs
nip44_conversation_key(merchant_id, peer_pubkey) -> bytes # for decrypt path only
nip44_decrypt(merchant_id, peer_pubkey, ciphertext) -> bytes
nip04_decrypt(merchant_id, peer_pubkey, ciphertext) -> bytes   # legacy compat
nip04_encrypt(merchant_id, peer_pubkey, plaintext) -> bytes    # NIP-15 replies
delete(merchant_id)
rewrap(old_version, new_version)                          # master-key rotation
```

Application services MUST NOT receive raw private keys. The decrypt path MAY take
ciphertext and return plaintext internally without exposing the key.

### 11.2 Local key backend (v1)

- Merchant nsec stored in `merchant_keys` table, encrypted with **AES-256-GCM**:
  `key_id || nonce(96-bit, random per encrypt) || ciphertext || tag`.
- Master key: 256-bit, provided via `GAMMAMARKET_MASTER_KEY` env (hex/base64) or
  LNbits secret source. If absent, merchant creation MUST fail loudly; keys MUST NOT
  be stored plaintext.
- AAD for the GCM envelope = `merchant_id || table || column` — prevents ciphertext
  transplant between merchants/fields.
- `key_id` identifies the master-key generation enabling rotation via `rewrap`.
- NIP-44 v2 and NIP-59 implemented **only** through `nostr-sdk` primitives; no custom
  secp256k1/chacha code.
- Decrypted message plaintext and shipping addresses: never logged, never returned
  on public routes, stored only in the encrypted fields defined in §11.3.

### 11.3 Sensitive-field encryption at rest

`orders.address_enc`, `orders.contact_json`, `inbox_events.raw_json` (when it
contains decrypted rumor content — store outer ciphertext only by default; store
decrypted rumor encrypted with the same envelope scheme under `field:` AAD) use the
same AES-256-GCM envelope with per-field AAD.

### 11.4 Public tokens

`public_token` = 256 bits from `secrets.token_urlsafe(32)`; stored as SHA-256 hash.
Comparison via `hmac.compare_digest`. Token rotation available on merchant request.

---

## 12. Configuration

| setting | default | notes |
|---|---|---|
| `GAMMAMARKET_MASTER_KEY` | — | required for key backend |
| `RESERVATION_TTL` | 900s | also invoice expiry |
| `OUTBOX_MAX_ATTEMPTS` | 20 | |
| `OUTBOX_BATCH` | 32 | rows per claim |
| `PEER_RELAY_TTL` | 24h | kind-10050 cache |
| `INBOX_MAX_EVENT_BYTES` | 32768 | pre-decode cap |
| `CHECKOUT_RATE_LIMIT` | 10/min/IP | §15 |
| `SPEC_REVISION` | `5dc79c5` | shown in settings UI |

---

## 13. Migration from `nostrmarket`

1. **Preview** builds a manifest: `{stalls, products, zones, quantities, ids, keys:
   pubkey-only}` — private keys are never exported by the catalog path.
2. **Execute** imports into canonical tables preserving the NIP-15 product id as
   `d_tag`; on `d_tag` collision, manifest marks the item `conflict` and the job
   cannot proceed until resolved.
3. **Dry run** renders all 30402/30405/30406/30017/30018 events and runs §6
   validators + golden fixture harness.
4. **Cutover** (explicit confirm): flips `publish` flags, enqueues all aggregates,
   then MUST enforce single-writer: either deactivate the `nostrmarket` merchant via
   its API, or require the operator to confirm deactivation manually — recorded in
   `migration_jobs.manifest_json.cutover_proof`. Both extensions writing the same
   addresses is a defect the migration MUST prevent, not warn about.

---

## 14. Idempotency, concurrency, multi-worker

- Idempotency is enforced by UNIQUE constraints (§4.13), not by check-then-insert.
- Every mutating public/admin endpoint accepts an optional `Idempotency-Key` header;
  replayed keys return the stored response.
- All order/reservation/settlement mutations run in single DB transactions; any
  cross-step failure rolls back fully.
- Multi-worker: task leases (§10), atomic outbox claims (§8.6), conditional stock
  updates (§8.2) are the only permitted concurrency primitives. No in-process locks
  may protect cross-request state.
- On startup: §8.7 reconciliation runs before subscriptions open.

---

## 15. Limits and rate limiting

| input | bound |
|---|---|
| raw inbox event JSON | 32 KB (pre-decode) |
| rumor content | 8 KB post-decrypt |
| tags per event | ≤ 128; tag element ≤ 2 KB |
| items per order | ≤ 64; qty per item ≤ 10⁶ |
| title/summary/description | ≤ 200 / 500 / 64 KB |
| images per product | ≤ 16; URL scheme `https` only; no server-side fetch |
| checkout POST | 10/min/IP + 100/hour/IP |
| open (unpaid) web orders per IP | ≤ 10 concurrent |
| public GETs | 120/min/IP |
| order DMs per buyer | 20/hour (inbound), excess → quarantine |

Markdown render (product descriptions, message content in UI) MUST be sanitized
(server-side allowlist renderer) — no raw HTML, no `javascript:`/`data:` URLs.

---

## 16. Observability

- Structured logs: `event=gammamarket.<component>.<action>` with ids/states; never
  keys, addresses, bolt11 strings in full (truncate to `payment_hash` correlation),
  or decrypted content.
- Metrics hooks: outbox depth/age, inbox depth, relay health per merchant,
  reservation count, stuck orders, failed publications.
- Merchant dashboard: per-relay status, outbox table, payment exceptions queue.

---

## 17. Test requirements

- **Golden fixtures** for every event in §6 — valid + invalid variants (bad sig,
  wrong kind, missing required tag, oversized, duplicate).
- **State-machine tests:** exhaustive legal/illegal transition table for §7.1–7.5.
- **Concurrency tests:** N parallel reservations against limited stock; duplicate
  settlement callbacks; duplicate gift-wrap delivery.
- **Security tests:** §21.3 checklist — seal/rumor pubkey mismatch, MAC failure,
  cross-merchant refs, token guessing, public amount manipulation, replayed payment
  events, log-scrubbing check (no nsec/address in emitted logs).
- **Interop:** at least one external Gamma client + one NIP-15 client exercised in
  manual conformance runs; results recorded per release.
- **Failure drills:** kill between invoice-paid and callback → reconciliation
  recovers; relay down → outbox retries → recovers.

---

## 18. Release gates

- **Release A** (catalog + web checkout): §6.1–6.8, §7.1–7.4, §8.1–8.4, 8.6–8.7,
  §11–§17 minus NIP-17 ingest. Must NOT claim order-protocol support.
- **Release B** (full Gamma merchant): + §8.5, §9.3, kind-10050 routing, type 1–4
  messages, kind-17 receipt handling.
- **Release C** (interop): + §6.6 NIP-15 events, §6.10 NIP-04 channel, §13 migration.

---

## 19. Threat model summary

Assets and controls per proposal §21; the normative requirements here:

- **Keys:** §11 envelope encryption; no plaintext persistence; no key material in
  API responses, logs, exceptions, or browser storage.
- **Payments:** LNbits state authoritative; receipts are hints; amount always
  server-computed; wallet ownership revalidated per invoice.
- **Inbound Nostr:** everything untrusted; verify-before-decrypt-before-dispatch
  order in §8.5 is mandatory; sizes bounded before decode.
- **Public API:** token-gated order views; rate limits; server-side totals;
  cross-merchant references rejected by FK + explicit check.
- **Privacy:** buyer address/contact encrypted at rest; NIP-04 channel labeled
  legacy; gift-wraps only to recipient relays (no metadata fan-out).

---

## 20. Traceability

Each section maps to proposal build phases: §4–§9 → Phases 1–3; §8.5/§9.3 → Phase 4;
§8.2–8.4/8.7 → Phase 5; §6.6/6.10 → Phases 6–7; §13 → Phase 8; §15–§17 → Phase 9.

---

## 21. Open questions / known gaps

These MUST be resolved or consciously deferred before Release B:

1. **Order-id authority:** the buyer chooses `order` tag value — a malicious buyer
   can squat order IDs or stuff huge/Unicode values. Spec bounds it (64 chars,
   `[A-Za-z0-9_-]`), but cross-merchant collisions are inherently possible; confirm
   uniqueness scope `(merchant, protocol, buyer, external_id)` is right.
2. **kind-17 receipt `amount` semantics:** spec says "payment amount" — ambiguous
   vs `total_sat`; we ignore it for settlement but must define display/dispute use.
3. **Refund path:** cancelled-after-confirm orders need an outgoing-payment
   permission model (LNbits admin key scope) — deferred to a later release; spec
   currently requires manual resolution only.
4. **`stock` tag truthfulness:** publishing `available` leaks reservation volume;
   alternatives (publish `on_hand`, or hysteresis) need a decision.
5. **Multiple merchants per user / shared wallets:** schema allows 1:1
   (`UNIQUE(user_id)`); multi-shop per user is a future change.
6. **NIP-89 handler identity:** 31990 signed by merchant key treats the extension as
   the app; a distinct app identity may be cleaner for marketplace-wide claims.
7. **`payment_preference` default:** market-spec defaults to `manual` but service-
   assisted mode wants a recommended app — our web checkout IS that app; confirm
   31989/31990 publication timing vs. merchant onboarding.
8. **Relay-side event replacement:** relays keep latest addressable event by
   `created_at`; signing with fresh timestamps assumes relay clock tolerance —
   define max-future skew handling if a relay rejects.
9. **Shipping `extra-cost` currency:** market-spec says "in the product's currency"
   but options have their own currency; define precedence when they differ.
10. **NIP-15 order `product_id` mapping:** NIP-15 buyers reference NIP-15 product
    ids; variation suffix mapping (§6.6) must round-trip through §8.1 — verify.
