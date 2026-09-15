# GammaMarkets WASM Extension for LNbits — Implementation Proposal

Status: proposal
Verified against: `~/github/lnbits` dev branch (`v1.6.2-rc1`, commit `e336fe1`)
Companion research: `research.md`

## 1. Goal

A standalone LNbits **WASM extension** that lets a merchant manage products once and
publish them as both:

- **NIP-15** marketplace events — stall `kind:30017`, product `kind:30018`
- **NIP-99 / GammaMarkets** events — classified listing `kind:30402`, collection
  `kind:30405`, shipping option `kind:30406`

…with Lightning checkout through LNbits invoices, public product pages, and
order tracking. No changes to the existing `nostrmarket` extension.

## 2. What the LNbits WASM runtime actually provides

Verified against `lnbits/core/wasm_ext/` on `dev` (available since v1.6.0):

| Capability | Mechanism | Notes |
|---|---|---|
| Storage | `ext.storage.read` / `.write` / `.read_public` / `.append_public` | KV-document model per owner, not SQL/migrations |
| Receive payments | `wallet.create_invoice`, `wallet.create_invoice_public`, `wallet.payments.watch`, `events.onInvoicePaid` | `create_invoice_public` is scoped by `{table, wallet_field}` policy — built for public checkout |
| Send payments | `wallet.pay_invoice`, `wallet.pay_invoice_background`, `wallet.pay_lnurl` | For refunds / affiliate payouts if needed |
| Outbound HTTP | `http.request` | **HTTPS only**, per-origin allowlist, SSRF-protected (no localhost/private IPs), 64KB max body |
| Call other extensions | `extension.api.request` | HTTP only, path must start `/api/`, per-extension `read`/`write` policies, requires user access token |
| Realtime to browser | `websocket.publish` / `websocket.subscribe` | Internal UI hub — **not** relay connections |
| Routes | `api_routes`, `ui_routes` with `auth: "public" \| "user"` | Public product page + checkout is supported |
| Events | `events.onInvoicePaid` only | **No cron/scheduler/timers** |
| Env/secrets | **none** | WASM cannot see `.env`; no secrets manifest field |
| Nostr | **none** | No `nostr.*` host API, no signing function |

### The one hard constraint: relay connectivity

Nostr relays speak WebSocket (`wss://`). The WASM runtime has **no outbound
WebSocket** and `http.request` is HTTPS-only.

`nostrclient` cannot bridge this gap: its Nostr protocol endpoint is itself a
WebSocket (`/nostrclient/api/v1/{ws_id}`), and its HTTP `/api/v1/` routes are
`check_admin`-gated relay management only — there is no HTTP "publish event"
endpoint. `extension.api.request` cannot open WebSockets and would not pass the
admin check with a regular user token anyway.

**Conclusion: a pure-WASM extension cannot publish to or subscribe to relays
from inside the sandbox today.** The design below routes around this.

## 3. Recommended architecture: sign in WASM, transport in the browser

```
┌──────────────────────────── LNbits host ────────────────────────────┐
│                                                                     │
│   gammamarket WASM module                                           │
│   ├── canonical product model                                       │
│   ├── NIP-15 event builder  (30017 stall, 30018 product)            │
│   ├── NIP-99/GM builder     (30402 listing, 30405 collection,       │
│   │                         30406 shipping)                         │
│   ├── schnorr signing       (merchant nsec from ext.storage)        │
│   ├── NIP-17/NIP-44 decrypt (incoming order DMs)                    │
│   ├── order state machine, inventory, outbox queue                  │
│   └── invoice glue (create_invoice_public, onInvoicePaid)           │
│                                                                     │
│   ext.storage: products / orders / outbox / settings                │
└──────────────▲───────────────────────────────────────▲──────────────┘
               │ api_routes (auth: user)               │ api/ui_routes (public)
               │                                       │
        merchant browser                        buyer browser
        ┌───────────────┐                         shows product page,
        │ UI JS relay   │                         invoice QR, pays
        │ transport:    │                         (no Nostr needed
        │ WSS → relays  │                          for checkout)
        └───────┬───────┘
                │ ["EVENT", {...}], ["REQ", ...]
                ▼
        Nostr relays (direct wss://)
        — or via nostrclient public WS if enabled (optional)
```

Key insight: **the extension's frontend JS runs in the merchant's browser, and
browsers can open WebSockets anywhere.** The WASM sandbox boundary only applies
to the module. So:

- The nsec stays server-side in `ext.storage` (never shipped to the browser).
- WASM builds + signs events; the response containing signed events goes to the
  UI; the UI pushes `["EVENT", ...]` over WSS.
- The UI subscribes (`["REQ", {kinds:[1059/4], #p: merchant_pubkey}, since:
  last_seen]`) when the merchant opens the orders page, and POSTs received
  ciphertexts to an authenticated `api_route`; WASM decrypts and creates orders.

### Why this beats the alternatives

| Option | Verdict |
|---|---|
| WASM → `extension.api.request` → nostrclient | **Impossible.** nostrclient's Nostr I/O is WS-only; its HTTP API is admin relay-CRUD. Would need an upstream `POST /api/v1/publish` endpoint (see §6). |
| WASM → `http.request` → relay | **Impossible.** HTTPS-only, no WS upgrade, and standard relays don't offer HTTP publish. |
| WASM → `http.request` → self-hosted bridge | Possible but adds a public HTTPS service + SSRF policy friction; worse than browser transport. |
| Host-mediated signing (`nostr.sign` host call, `.env` nsec) | **Doesn't exist.** No such host API or secrets mechanism. |
| Browser transport (recommended) | Zero core changes, no extension dependencies, key stays server-side. |

### The accepted trade-off

Relay I/O only happens while a merchant has the UI open (plus one
server-side path, below). For a marketplace this is acceptable:

- Listings change rarely; merchant publishes when editing products anyway.
- Order DMs are picked up via `since: last_seen` sync whenever the merchant
  opens the app — like an email client, not a server daemon.
- Buyers don't need Nostr at all — checkout is a plain LNbits web page.

### Server-side publish where it matters: the outbox

When `onInvoicePaid` fires (e.g. last item sold → `status: "sold"`), no browser
may be open. WASM writes the republish payload to an `outbox` storage table.
Next time the merchant opens the UI, it drains the queue. Optional upgrade path
in §6 removes this limitation.

## 4. Detailed design

### 4.1 `config.json` manifest (real permission IDs only)

```json
{
  "id": "gammamarket",
  "name": "GammaMarkets",
  "short_description": "NIP-15 + NIP-99 dual-protocol marketplace",
  "version": "0.1.0",
  "min_lnbits_version": "1.6.0",
  "extension_type": "wasm",
  "wasm": {
    "module": "gammamarket.wasm",
    "exports": [
      {"name": "save_product",        "visibility": "authenticated"},
      {"name": "build_events",        "visibility": "authenticated"},
      {"name": "ingest_nostr_event",  "visibility": "authenticated"},
      {"name": "get_product_public",  "visibility": "public"},
      {"name": "create_checkout",     "visibility": "public"},
      {"name": "on_invoice_paid",     "visibility": "event"}
    ]
  },
  "events": {"onInvoicePaid": "on_invoice_paid"},
  "ui_routes": [
    {"path": "/",              "entrypoint": "admin.html",   "auth": "user"},
    {"path": "/p/{productId}", "entrypoint": "product.html", "auth": "public",
     "path_params": {"productId": "product_id"}}
  ],
  "api_routes": [
    {"method": "POST", "path": "/api/v1/products",   "export": "save_product",       "auth": "user"},
    {"method": "POST", "path": "/api/v1/publish",    "export": "build_events",       "auth": "user"},
    {"method": "POST", "path": "/api/v1/inbox",      "export": "ingest_nostr_event", "auth": "user"},
    {"method": "GET",  "path": "/api/v1/p/{productId}", "export": "get_product_public", "auth": "public",
     "path_params": {"productId": "product_id"}},
    {"method": "POST", "path": "/api/v1/checkout",   "export": "create_checkout",    "auth": "public"}
  ],
  "permissions": [
    {"id": "ext.storage.read"},
    {"id": "ext.storage.write"},
    {"id": "ext.storage.read_public",  "policies": [
      {"table_name": "products", "public_fields": ["title","summary","price","currency","images","status","shipping","d_tag"]}
    ]},
    {"id": "wallet.create_invoice"},
    {"id": "wallet.create_invoice_public", "policies": [
      {"table": "products", "wallet_field": "wallet_id"}
    ]},
    {"id": "wallet.payments.watch"},
    {"id": "websocket.publish", "policies": [{"max_messages_per_second": 10}]},
    {"id": "utils.basic"}
  ]
}
```

Note: permissions are validated strictly — any ID not in the host registry
fails install. All permission requests must be **granted by an admin** at
install time (`granted_permissions` flow in `wasm_ext/api/permissions.py`).

### 4.2 Storage model (`ext.storage` tables)

| Table | Key fields |
|---|---|
| `settings` | `nsec` (server-only), `pubkey`, `wallet_id`, `relays[]`, `nip15_stall_id`, `last_seen` |
| `products` | `id`, `d_tag` (= nip15 product id), `wallet_id`, title, price, currency, inventory, status, images, shipping_refs |
| `collections` | `d_tag`, product ids → `kind:30405` |
| `orders` | `id`, product_id, buyer pubkey, payment_hash, state, NIP-17 thread refs |
| `outbox` | signed events pending browser publish (sold-out updates etc.) |
| `inbox_log` | processed event ids (dedupe) |

One canonical product → both representations, same `d` tag, so updates replace
addressable events instead of duplicating.

### 4.3 Event construction

WASM holds a small Nostr library (Rust → wasm32, or TinyGo/AssemblyScript):

- `kind:30018` NIP-15 product ← product fields + stall ref
- `kind:30017` stall ← merchant profile/stall config
- `kind:30402` NIP-99 listing ← same product, `d` tag = product id,
  GammaMarkets tags: `price`, `stock`, `shipping`, `payment`, `t` categories
- `kind:30405` collection, `kind:30406` shipping options
- Signing: BIP-340 schnorr over `sha256([0,pubkey,created_at,kind,tags,content])`
- Inbound: NIP-17 gift-wrap unwrap + NIP-44 decrypt (secp256k1 ECDH + ChaCha20)
  — pure crypto, no host calls needed

### 4.4 Flows

**Publish (merchant action)**
```
UI: POST /api/v1/publish {product_ids}
 → WASM: load products + nsec → build+sign 30017/30018/30402/30405/30406
 → response: {events: [...]}
UI: for each relay: ws.send(["EVENT", ev]) → collect OK
```

**Buyer checkout (public, no Nostr)**
```
GET  /gammamarket/p/{id}      → product.html → GET /api/v1/p/{id}
                                (storage.get_public)
POST /api/v1/checkout {id}    → wallet.create_invoice_public
                                (wallet resolved from products.wallet_id)
                                → order row: state=pending
product.html polls /api/v1/p/{id} or receives websocket.push
```

**Payment**
```
LNbits → onInvoicePaid (extra.source_id → owner resolved via products table)
 → WASM: order.state=paid, inventory--, build status update events → outbox
 → websocket.publish → checkout page flips to "paid"
```

**Inbound orders/DMs**
```
Merchant opens UI → WSS REQ since:last_seen → ciphertexts
 → POST /api/v1/inbox → WASM: unwrap NIP-17, parse order, store, reply events
 → UI publishes replies
```

### 4.5 Security notes

- nsec in `ext.storage` is the same exposure as `nostrmarket` (which stores
  merchant keys in its DB). Never returned by any public export; never sent to
  frontend — only signed events leave the module.
- `wallet.create_invoice_public` is tightly scoped: buyers can only create
  invoices for wallets referenced by product rows. Rate-limit the public
  checkout route at the LNbits/reverse-proxy layer.
- All requested permissions are visible to the admin at install — keep the
  manifest minimal (no `wallet.pay_invoice` unless shipping refunds).

## 5. WASM runtime limits — what to raise and why

Defaults (`settings.py`, `WasmRuntimeLimits`) vs. suggested values for this
extension. All are admin-tunable via env vars / the WASM admin UI
(`wasm-runtime` / `wasm-limit-config` admin components) and can be overridden
per installed extension.

| Setting | Default | Suggested | Why |
|---|---|---|---|
| `wasm_runtime_max_execution_ms` | 5 000 | 15 000–30 000 | Catalog import / batch re-signing (N products × 2–3 events). Signing is fast (~ms), so this is mostly headroom for big catalogs. |
| `wasm_runtime_max_host_calls` | 1 000 | 3 000–5 000 | One storage read per product during batch ops. |
| `wasm_runtime_max_storage_calls` | 100 | 300–500 | Same — products + collections + outbox writes per publish run. |
| `wasm_runtime_max_memory_bytes` | 64 MB | 64–128 MB | Schnorr/NIP-44 are light; bump only if embedding a heavy SDK. |
| `wasm_runtime_max_fuel` | 100 M | keep / 250 M | Crypto is fuel-hungry; measure, then decide. |
| `wasm_runtime_max_http_calls` | 20 | keep | Irrelevant with browser transport; would matter only for a companion-endpoint publish path (≤20 relay calls per invocation → chunk). |
| `wasm_runtime_max_concurrent_invocations_per_user` | 4 | keep | Checkout is one call per purchase; fine. |
| `wasm_runtime_max_response_bytes` | 1 MB | keep | Batch of signed events ≪ 1 MB for realistic catalogs. |

Env override pattern: `LNBITS_WASM_RUNTIME_MAX_EXECUTION_MS=30000` etc.

**Design rule regardless of limits:** chunk batch work. Publish/import in
pages of ~50 products per invocation rather than raising limits to fit a
monolithic loop — keeps the extension healthy on stock configurations too.

## 6. Does it need `nostrclient`?

**No — and it can't use it today anyway.** Summary:

- `extension.api.request` → nostrclient: dead end (nostrclient's Nostr I/O is
  WebSocket-only; its `/api/` HTTP routes are admin relay management).
- Browser JS → nostrclient public WS (`/nostrclient/api/v1/relay`, requires
  `public_ws: true` in its config): **optional** convenience — reuse the
  admin's centrally managed relay pool instead of per-extension relay config.
  Worth supporting as a config toggle, not a dependency.

**Optional server-side upgrade path** (removes the "UI must be open" caveat):

Add `POST /nostrclient/api/v1/publish` upstream — a user-scoped (not
`check_admin`) endpoint accepting a signed event and broadcasting it through
nostrclient's existing `relay_manager`. ~30 lines. Then:

```json
{"id": "extension.api.request", "policies": [{"id": "nostrclient", "access": ["write"]}]}
```

lets WASM publish from `onInvoicePaid` with no browser open, draining the
outbox server-side. Inbound subscriptions would additionally need a
poll/fetch endpoint (`GET /api/v1/events?since=...`) — same pattern. Until
such an endpoint exists, the browser-transport + outbox design is fully
self-sufficient.

## 7. Build plan

1. **Skeleton**: Rust (or AssemblyScript) module; `config.json`; one
   authenticated `api_route` + one `ui_route`; storage round-trip. Verify on
   local dev LNbits (`make dev`).
2. **Nostr core**: event builders for 30017/30018/30402/30405/30406, schnorr
   signing, unit tests against published test vectors.
3. **Merchant UI**: product CRUD, relay list config, nsec input (stored via
   `ext.storage.write`, displayed once), "Publish" button → browser WSS.
4. **Checkout**: public product page, `create_invoice_public`, `onInvoicePaid`
   → order paid + outbox + `websocket.publish` status push.
5. **Inbox**: UI-side REQ sync, `ingest_nostr_event`, NIP-17 decrypt, order
   list UI, reply publish.
6. **Polish**: outbox drain on UI load, import from `nostrmarket` (reads its
   product data only if a compatible API exists — otherwise CSV/JSON import),
   docs, registry submission (`extensions.wasm.json` is currently empty —
   this would be among the first WASM extensions published).

## 8. Open questions / risks

- **nostrclient publish endpoint** — worth upstreaming; removes the only real
  limitation. Confirm appetite with lnbits/nostrclient maintainers.
- **Event-context auth**: verify which host calls are legal inside
  `onInvoicePaid` (context `event`, resolved `owner_id`). `create_invoice`
  isn't needed there, but `ext.storage` writes and `websocket.publish` are —
  test early.
- **Rate limiting** on public routes: checkout spam creates real invoices.
  LNbits has middleware-level controls; document recommended config.
- **GammaMarkets spec is a draft** — tag conventions may shift; isolate spec
  mapping in one module.
- **Name**: "GammaMarkets-compatible" in description is safer than claiming
  the name unless maintainers endorse it.
