# Conversation Transcript

## User

is NIP-99  standard in LNBITS?

## Assistant

Not currently, at least not as a built-in LNbits marketplace standard.

LNbits’ **Nostr Market** extension is based on **NIP-15**, using events such as:

- `30017` — stall
- `30018` — product
- `30019` — marketplace configuration

It does not natively publish products as **NIP-99 Classified Listings** (`kind: 30402`). Support for adding NIP-99 publishing to the LNbits `nostrmarket` extension has been proposed, but the available implementation references indicate it is not yet part of the standard extension. <citation src="8,6"></citation>

You could add NIP-99 support through a custom LNbits extension or by publishing `kind:30402` events separately, while continuing to use LNbits for Lightning invoices and payments.

## User

why wasn't NIP-99 ever added to LNBits? was there any discussion historically on the github about NIP-99?

## Assistant

Yes—there was a GitHub discussion, but it was very limited and does **not** document a technical rejection by LNbits maintainers.

The relevant item is **`lnbits/nostrmarket` issue #144**, titled “Support publishing products as NIP-99 Classified Listings.” It was opened on **December 22, 2025** and proposed:

- Publishing products as `kind:30402`
- Mapping existing LNbits product fields to NIP-99/GammaMarkets tags
- Optionally publishing both NIP-15 and NIP-99 events
- Improving compatibility with Shopstr, Cypher, Plebeian, Conduit, and other clients

However, GitHub currently shows the issue as **closed**, with **no assignee, no linked branch or pull request, and no visible maintainer discussion explaining why it was closed**. So there is no public evidence that the feature was rejected for a specific reason—or that it was actually implemented. <citation src="1"></citation>

The most likely explanation is architectural rather than ideological:

1. **LNbits’ NostrMarket extension was built around NIP-15.**  
   Its data model and workflow use stalls, products, and a structured order/payment/status exchange. NIP-15 defines not just product listings but also the marketplace relationship and checkout protocol. <citation src="7"></citation>

2. **NIP-99 is not a direct replacement for the whole NIP-15 workflow.**  
   NIP-99 primarily defines a self-contained classified-listing event, `kind:30402`, with metadata such as title, summary, location, price, and status. It does not by itself provide the same structured stall/order/payment machinery. <citation src="13"></citation>

3. **Adding it would require deciding how the two models coexist.**  
   The simplest approach would be dual publishing: retain NIP-15 for LNbits’ existing checkout flow while publishing a corresponding NIP-99 listing for discovery. But that raises synchronization questions—product IDs, inventory, price changes, sold status, images, shipping, and whether an NIP-99 buyer should enter the existing NIP-15 order flow.

4. **The specification landscape moved after LNbits had already committed to NIP-15.**  
   The official NIPs repository now labels NIP-15 “unrecommended” because it is too complicated and points implementers toward NIP-99. But that appears to be a later ecosystem development, not evidence that LNbits deliberately refused NIP-99 earlier. <citation src="12,13"></citation>

So the careful conclusion is:

> NIP-99 was not historically ignored altogether—there is at least one explicit feature request—but the GitHub record does not show a substantive design debate or a maintainer explanation for its closure. The available evidence suggests that NIP-99 simply never received an implementation owner or pull request, while the extension remained centered on its original NIP-15 architecture.

Also, strictly speaking, NIP-99 is an optional Nostr implementation possibility, not a requirement that every Nostr or Lightning application must support. <citation src="12"></citation>

## User

Would it be possible to make a LNBIts extension for NIP-99 that would work with the NIP-15 nostrmarket to just completely separate extension that does both NIP-15+NIP-99 events, as a WASM extension? Maybe give it an alternative name called GammaMarket WASM extension?

## Assistant

Yes. A standalone LNbits extension could publish **both NIP-15 and NIP-99/GammaMarkets events** without modifying the existing `nostrmarket` extension. That is probably the cleanest way to experiment with dual-protocol support.

The important distinction is that it would be a **separate extension**, not an extension “inside” the existing NIP-15 one:

- Existing `nostrmarket` remains the NIP-15 merchant and checkout system.
- The new extension maintains its own product/listing database.
- It publishes:
  - NIP-15 stall/product events, such as `30017` and `30018`
  - NIP-99 classified listings, `30402`
- Both events point to the same internal product record and LNbits wallet.

The existing Nostr Market extension already uses `nostrclient`, merchant keys, stalls, products, shipping zones, and NIP-04-based order/payment messaging. It also treats NIP-15 events as the source of the marketplace structure. <citation src="7"></citation>

## The main technical challenge

NIP-99 is primarily a **listing and discovery format**. It does not provide the complete shopping-cart, order, invoice, and fulfillment protocol that NIP-15 provides. The NIP-99 specification itself describes NIP-15 as the more strictly structured marketplace protocol and NIP-99 as the simpler, more flexible alternative. <citation src="13"></citation>

Therefore, the new extension should not treat the two event types as independent products. It should use one canonical product model and generate two representations:

```text
LNbits product
├── NIP-15 representation: stall/product/order flow
└── NIP-99 representation: public classified listing
```

A practical purchase flow would be:

1. A buyer discovers the product through a `30402` NIP-99 event.
2. The listing contains the price, currency, stock/status, images, shipping information, and merchant identity.
3. The buyer is directed to either:
   - the LNbits NIP-15 marketplace checkout, or
   - a new checkout endpoint in the dual-protocol extension.
4. The extension creates an LNbits invoice.
5. Payment status is monitored through LNbits.
6. The product’s NIP-15 and NIP-99 events are republished with updated stock or `status` values.

NIP-99 already standardizes fields such as `title`, `summary`, `published_at`, `location`, `price`, and `status`; GammaMarkets adds e-commerce conventions for pricing, shipping, quantity, payment methods, categories, and tags. <citation src="13,6"></citation>

## Recommended architecture

### 1. Make the new extension the source of truth

Do not have the new extension scrape or duplicate the existing NIP-15 database. Give it its own schema, for example:

```text
ext_gammamarket/
├── merchants
├── products
├── listings
├── inventory
├── orders
├── payments
└── relay_configs
```

LNbits’ extension model supports separate routers, database namespaces, migrations, API endpoints, frontend templates, and invoice listeners. <citation src="4,2"></citation>

Each product should have stable identifiers:

```text
internal_product_id
nip15_product_id
nip99_d_tag
merchant_pubkey
wallet_id
```

The same stable product ID should be used where possible in both event representations. The NIP-99 event would use a stable `d` tag so updates replace the previous listing rather than creating duplicates.

### 2. Publish NIP-15 and NIP-99 together

For each product update:

```text
save product
  ↓
publish NIP-15 stall/product events
  ↓
publish NIP-99 kind:30402 event
```

The extension should expose separate toggles:

- Publish NIP-15
- Publish NIP-99
- Publish to both
- Publish only to selected relays

This allows a merchant to use the extension as a normal NIP-15 shop, a NIP-99 listing publisher, or both.

### 3. Keep checkout explicit

A NIP-99 listing should not imply that every client can complete checkout. Some clients may only display the listing.

The event could include GammaMarkets-compatible payment and commerce tags, plus a link to a checkout page or product endpoint. But the extension should also provide a clear web checkout route, for example:

```text
https://example.com/gammamarket/product/<id>
```

That page can:

- display current stock and price;
- calculate shipping;
- create an LNbits invoice;
- show a Lightning QR code;
- confirm payment;
- create the order.

This is more reliable than assuming a random NIP-99 client understands LNbits invoices or the NIP-15 order protocol.

### 4. Synchronize status carefully

The extension would need to republish both event types when any of these change:

- price;
- currency;
- inventory;
- product availability;
- shipping cost;
- images;
- title or description;
- sold status.

For NIP-99, that means updating the same `30402` addressable event with the same `d` tag. For NIP-15, it means republishing the corresponding product/stall event. NIP-99 supports active/sold status, while GammaMarkets adds conventions for inventory and commerce metadata. <citation src="13,6"></citation>

## About WASM

This is the part that needs clarification: current LNbits extension documentation describes extensions as **Python backend code with API routes, database models, migrations, and JavaScript/Vue frontend code**. It does not describe a general WebAssembly extension runtime. <citation src="4,2"></citation>

There is a repository named `lnbits-extensions-wasm`, but its description identifies it as a **registry of extensions that do not need vetting**, not as a WebAssembly execution API for LNbits extensions. <citation src="1"></citation>

So:

- A normal LNbits extension can absolutely do this.
- A browser-side WASM module could potentially be used for isolated frontend logic.
- A pure WASM backend extension would require LNbits core support for loading WASM modules, exposing database and wallet capabilities, handling permissions, and registering routes.
- Without that runtime, calling it a “WASM extension” would likely mean a normal Python LNbits extension that happens to include WASM code, not a native WASM plugin.

For a first implementation, I would use the standard Python LNbits extension architecture and avoid WASM unless you specifically need sandboxing or want to share protocol code with another WASM-capable application.

## Naming

“GammaMarket” or “GammaMarkets” would communicate the intended protocol, but it could imply that the extension is officially maintained or endorsed by the GammaMarkets group. The existing LNbits feature request itself calls the proposal the “GammaMarkets spec,” so the association is real, but endorsement should not be assumed. <citation src="6"></citation>

Safer names would be:

- **Nostr Market Dual Publisher**
- **LNbits NIP-99 Market**
- **Nostr Commerce**
- **Nostr Market Interop**
- **GammaMarkets-compatible LNbits extension**

If the GammaMarkets maintainers approve the name, **GammaMarket LNbits Extension** would be understandable. Otherwise, “GammaMarkets-compatible” in the description is safer than using GammaMarkets as the main product name.

The most viable MVP would therefore be:

1. standalone standard LNbits extension;
2. own product/order tables;
3. merchant key and relay configuration;
4. NIP-15 + NIP-99 dual publishing;
5. LNbits invoice creation and payment listener;
6. product page for checkout;
7. inventory/status synchronization;
8. optional import of existing NIP-15 products from `nostrmarket`.

That would provide interoperability without risking regressions in the existing NIP-15 extension.

## User

is NIP-99 and Gamma Markets synonymous? here's the spec: https://github.com/GammaMarkets/market-spec/blob/main/spec.md

## Assistant

No. **NIP-99 and GammaMarkets are related, but they are not synonymous.**

The relationship is:

```text
NIP-99
└── GammaMarkets e-commerce extension
    ├── product listings
    ├── collections
    ├── shipping
    ├── orders
    ├── payments
    └── reviews
```

**NIP-99** is the base Nostr specification for classified listings. Its core requirement is an addressable `kind:30402` event describing something offered for sale or otherwise available. It supports broad use cases—including products, services, jobs, rentals, giveaways, and personal listings—and only defines lightweight listing metadata such as title, summary, price, location, images, and status. <citation src="3"></citation>

**GammaMarkets** is an e-commerce protocol built on top of that NIP-99 foundation. Its specification explicitly requires `kind:30402` product listings, but also adds a larger set of marketplace mechanisms:

- `kind:30405` product collections
- `kind:30406` shipping options
- product variations and inventory
- merchant application preferences
- NIP-17 encrypted order communication
- payment requests and receipts
- order-status updates
- shipping updates
- product reviews

The GammaMarkets specification labels itself a **draft** and describes these as required or optional components of a full marketplace implementation. <citation src="1"></citation>

So the most accurate terminology would be:

- A client that only publishes `kind:30402` events: **NIP-99 client**
- A client implementing the GammaMarkets e-commerce rules: **GammaMarkets-compatible client**
- A client doing both: **NIP-99/GammaMarkets-compatible client**
- A client implementing GammaMarkets: it necessarily uses NIP-99 product listings, but NIP-99 clients do not necessarily implement GammaMarkets checkout or order handling

This also corrects the earlier simplification: GammaMarkets is not merely a set of extra NIP-99 tags. It is a broader marketplace protocol whose product listing layer uses NIP-99.

For the proposed LNbits extension, the clearest description would be:

> **An LNbits GammaMarkets-compatible marketplace extension that publishes NIP-99 product listings and supports GammaMarkets order/payment flows.**

That would be more precise than calling it simply an “NIP-99 extension.” A minimal version could publish only `30402` listings, while a full version would also implement collections, shipping, NIP-17 order messages, payment requests, receipts, and status updates.

## User

would this extension work in the WASM sandbox?

## Assistant

**Not as a pure WASM module under the currently documented LNbits extension model.** It could work only if the WASM sandbox exposes host APIs for storage, Lightning invoices, Nostr relay access, signing, and background tasks.

LNbits extensions are currently designed as plugins that can:

- register API routes;
- create database tables;
- serve a UI;
- listen for invoice events;
- run background tasks.

Those capabilities are provided by the normal LNbits extension environment, not automatically by WebAssembly. <citation src="6"></citation>

The `extensions.wasm.json` registry should not be interpreted as proof that LNbits can execute arbitrary WASM extensions. The registry repository describes itself as a registry for extensions that do not need the normal vetting process; it does not document a general WASM plugin ABI or sandbox runtime. <citation src="2"></citation>

## What would work

A practical design would be a **hybrid extension**:

```text
LNbits Python host extension
├── database and migrations
├── LNbits invoice/payment API
├── Nostr relay connections
├── key management and signing
├── background synchronization
└── WASM protocol module
    ├── NIP-15 event construction
    ├── NIP-99 event construction
    ├── GammaMarkets tag validation
    ├── serialization
    └── deterministic compatibility tests
```

The WASM component could safely handle deterministic protocol logic:

- generate NIP-15 stall/product event payloads;
- generate NIP-99 `kind:30402` events;
- validate GammaMarkets tags;
- map one canonical product object into both event formats;
- validate orders and status transitions;
- calculate hashes or canonical representations.

The host extension would still need to handle:

- LNbits wallet authentication;
- invoice creation;
- payment webhooks or invoice listeners;
- database access;
- Nostr relay WebSockets;
- encrypted order messages;
- merchant private-key protection;
- scheduled republishing when stock or payment status changes.

A fully isolated WASM sandbox with no network, filesystem, clock, environment, or host imports could not perform the complete marketplace job by itself. It could construct an event, but it could not publish that event, create a Lightning invoice, receive payment notifications, or persist orders.

## The key security issue

Do not put the merchant’s Nostr private key or LNbits admin key directly into an untrusted WASM module.

A better capability design would be:

```text
WASM requests:
  sign(event_hash)

Host:
  checks allowed event kind and merchant scope
  signs with the protected Nostr key
  returns the signature
```

Likewise, the WASM code should request:

```text
create_invoice(amount, memo, order_id)
```

rather than receiving an unrestricted LNbits admin key.

## Best implementation path

For an initial release, I would build it as:

1. a normal Python LNbits extension;
2. a protocol library that can optionally compile to WASM;
3. a web UI using the normal LNbits extension frontend;
4. WASM-based validation/event-generation tests;
5. a future host ABI if LNbits later supports real WASM extensions.

That gives you a working GammaMarkets-compatible extension today while keeping the protocol layer portable to WASM later.

The most accurate description would therefore be:

> **A standard LNbits extension with a WASM-compatible GammaMarkets/NIP-15/NIP-99 protocol core**

—not a pure “GammaMarket WASM extension,” unless the target LNbits deployment specifically provides a documented WASM runtime and host API.

## User

All LNBits versions post v1.6.0 have the ability to deploy wasm extensions

## Assistant

You’re right—I was wrong about the WASM capability. **LNbits versions after v1.6.0 can deploy WASM extensions**, and the `paidtasks` repository is an official example of that model. It uses a WASM module, `config.json` permissions, database access, payment creation, and payment watchers. <citation src="1,2"></citation>

So yes: a **GammaMarkets-compatible NIP-15 + NIP-99 extension can be built for the LNbits WASM sandbox**.

The design would look like this:

```text
GammaMarket WASM extension
├── WASM module
│   ├── product/listing logic
│   ├── NIP-15 event generation
│   ├── NIP-99 kind:30402 generation
│   ├── GammaMarkets validation
│   ├── order state machine
│   └── inventory logic
├── public handlers/UI
└── declared LNbits permissions
    ├── database read/write
    ├── create incoming invoices
    ├── payment watching
    └── any permitted Nostr/network capability
```

The official example shows that a WASM extension can declare database permissions, call the payment API with create-only restrictions, watch payments, and use payment tags for user authorization. <citation src="2"></citation>

The main implementation question is **how the extension publishes and signs Nostr events**.

There are three possible approaches:

1. **Host-provided Nostr API**  
   Best option. The WASM extension requests something like:

   ```text
   nostr.publish(event)
   nostr.sign(event_hash)
   ```

   The LNbits host handles relay connections and protects the merchant’s private key.

2. **Permitted internal LNbits endpoint**  
   The extension calls a Nostr-publishing service through an allowed LNbits API route. Any internal endpoint would need to be explicitly declared in `config.json`; the WASM example specifically requires internal endpoint access to be declared. <citation src="2"></citation>

3. **Direct relay access from WASM**  
   This is possible only if the sandbox grants outbound network access or provides a relay proxy. If arbitrary WebSocket/network access is blocked, the module cannot directly publish to Nostr relays.

For security, the third-party extension should not receive a raw Nostr private key. The preferred model is capability-based signing:

```text
WASM → request signature for approved event
LNbits host → verify permissions and event kind
LNbits host → sign with protected merchant key
LNbits host → publish to configured relays
```

The extension would need permissions approximately equivalent to:

```json
{
  "permissions": [
    "ext.db.read_write",
    "ext.payments.watch",
    "api.POST:/api/v1/payments",
    "nostr.publish",
    "nostr.sign"
  ]
}
```

The exact Nostr permission names would depend on the current LNbits WASM host API; they should be taken from the version’s WASM documentation rather than invented in the manifest.

A good first version would implement:

- product creation and editing;
- NIP-99 `kind:30402` product listings;
- NIP-15 `30017`/`30018` compatibility events;
- GammaMarkets collections and shipping metadata;
- LNbits invoice creation;
- payment watching;
- stock and sold-status updates;
- a public product page and checkout;
- relay configuration;
- stable mapping between one product and both event formats.

The extension should use one canonical internal product record:

```text
product_id
nip15_product_id
nip99_d_tag
merchant_pubkey
price
currency
inventory
status
shipping_options
```

Then every product change republishes both representations. The NIP-99 event provides discovery and GammaMarkets compatibility, while the NIP-15 representation can preserve compatibility with existing Nostr Market clients.

So the corrected answer is:

> **Yes, this is a realistic LNbits WASM extension.** The only part that must be confirmed against the target LNbits version is whether the WASM host already exposes Nostr relay publishing/signing. If it does not, the extension would need either a small host-side Nostr capability or a permitted relay-proxy endpoint.

## User

could a wasm extension use a nostr private key from the LNBits .env file?

## Assistant

**Technically, yes—but not automatically, and it should not be exposed to the WASM module as a normal environment variable.**

LNbits loads its `.env` configuration in the host process. A WASM extension runs inside the WASM sandbox, so it cannot simply call something like:

```rust
std::env::var("NOSTR_PRIVATE_KEY")
```

unless the LNbits WASM runtime explicitly exposes environment variables to the module. Exposing the entire process environment would be unsafe because it could include database credentials, authentication secrets, wallet configuration, and other sensitive values. LNbits’ own configuration includes secrets such as `AUTH_SECRET_KEY`, and wallet admin keys must be treated as highly sensitive credentials. <citation src="1,3"></citation>

A better implementation is for LNbits to expose a **specific secret or signing capability** to the extension.

### Preferred design: host-mediated signing

The `.env` could contain a merchant key:

```env
GAMMAMARKET_NOSTR_NSEC=...
```

But the WASM module should not receive the raw `nsec`. Instead:

```text
WASM extension
  → sends unsigned Nostr event or event hash
LNbits host
  → reads GAMMAMARKET_NOSTR_NSEC
  → signs the event
  → returns the signature
WASM extension
  → publishes or requests publication
```

Even better, the host could expose:

```text
nostr.get_public_key()
nostr.sign_event(event)
nostr.publish_event(event)
```

The extension would then have no access to the private key at all.

### If the sandbox supports secret injection

If the WASM runtime has a declared secret-permission mechanism, the extension could request only one named secret, such as:

```json
{
  "secrets": ["GAMMAMARKET_NOSTR_NSEC"]
}
```

That would be workable, but the permission should be:

- explicit in the extension manifest;
- visible to the LNbits administrator;
- limited to that one variable;
- unavailable to frontend code;
- excluded from logs and error messages;
- unavailable to arbitrary user-supplied handlers.

Whether this exact manifest format exists depends on the WASM host API in the particular LNbits version. The `.env` file itself does not make variables available to an extension automatically.

### Important distinction

A Nostr private key can generally sign events, but it does **not** grant access to an LNbits wallet. The extension would still need separate LNbits permissions for:

- creating invoices;
- observing payment status;
- reading or writing extension data;
- sending event data to Nostr relays.

For the GammaMarkets extension, I would use:

```text
.env:
  GAMMAMARKET_NOSTR_NSEC

WASM:
  product and order logic
  NIP-15/NIP-99 event construction
  GammaMarkets validation

LNbits host:
  private-key storage
  Nostr signing
  relay publishing
  invoice creation
  payment watchers
```

So the answer is **yes, the extension can use a Nostr private key configured through `.env` if the LNbits WASM host provides a controlled secret/signing interface**. It should not receive the complete `.env` or read the raw private key directly.

## User

why does the lnbits nostrmarket extension depend on the nostr client extension and any other lnbits extensions? or is this me being incorrect? please check

## Assistant

You’re **mostly correct**: the current LNbits **Nostr Market** extension explicitly requires the **`nostrclient` extension**, but it does not appear to require other LNbits extensions for its core operation. <citation src="1,2"></citation>

The dependency exists because `nostrmarket` delegates Nostr relay connectivity to `nostrclient`.

`nostrclient` acts as a relay multiplexer:

```text
Nostr Market
     │
     │ one WebSocket connection
     ▼
nostrclient
     │
     ├── Relay 1
     ├── Relay 2
     ├── Relay 3
     └── Relay 4
```

It maintains persistent connections to configured relays, fans out subscriptions and events, aggregates responses, deduplicates events, and reconnects when necessary. The Nostr Market code has a client specifically described as connecting to the `nostrclient` extension for relay communication. <citation src="3"></citation>

Nostr Market needs that service for:

- publishing stalls and products;
- receiving orders and encrypted messages;
- handling NIP-04 customer communication;
- receiving Nostr events;
- republishing products;
- maintaining a persistent relay connection.

The Nostr Market README therefore lists `nostrclient` as a prerequisite and says it must be installed, activated, and configured. <citation src="2"></citation>

The other apparent dependencies are different:

- **LNbits core:** database access, wallet/payment APIs, authentication, invoice listeners, and extension routing. This is not another optional extension.
- **A wallet/funding source:** required by LNbits to receive Lightning payments, but not necessarily a separate extension.
- **User Manager:** useful for creating users and assigning wallets on some deployments, but not required by the Nostr Market code itself.
- **NIP-15/Nostr clients:** external clients, not LNbits extensions.

Nostr Market also handles several things internally: merchant keypairs, NIP-15 event construction, NIP-04 encryption, stalls, products, zones, orders, invoices, and customer records. Its documentation says order placement, invoicing, payment details, and order status are handled through the Nostr Market implementation and NIP-04 messaging. <citation src="2,3"></citation>

So the dependency is best summarized as:

```text
Required:
  LNbits core
  nostrclient extension
  configured Lightning wallet/backend

Optional:
  User Manager
  other marketplace/client extensions
  external Nostr clients
```

For a new WASM GammaMarkets extension, you could either:

1. **Depend on `nostrclient`**, using it as the relay multiplexer; or
2. **Make the WASM extension self-contained**, if the WASM host provides outbound WebSocket/network access and suitable signing/storage/payment capabilities.

Using `nostrclient` would be the more compatible option for an LNbits deployment, because it avoids duplicating relay management. However, a WASM extension would need a supported way to call the `nostrclient` endpoint or an LNbits host capability that proxies Nostr operations. The existing Nostr Market dependency is therefore primarily a **relay transport and connection-management dependency**, not a dependency for NIP-15 logic itself.
