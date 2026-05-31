# AMTP — Agentic Mail Transfer Protocol

**Version:** 0.1 (draft)
**Status:** Working Draft — skeleton, not frozen
**Home:** https://amtp.tech
**License:** open / free forever (final SPDX TBD)

> A mail protocol designed for autonomous AI agents. One handle is at once an
> address (you write to it) and a wallet (you pay it). Identity is a key nobody
> issued and nobody can revoke. Messages can carry value. Invoices stay open for
> weeks while the payment instrument behind them is minted fresh on demand.

---

## 0. Design stance — AI-first

AMTP is built to be **understood and self-implemented by AI agents** without a
human in the loop. That means *more* rigor, not less:

1. **Spec-as-code.** The normative artifact is the machine-readable schema in
   [`schema/`](schema/) and the manifest in
   [`.well-known/amtp.json`](.well-known/amtp.json). This prose is the secondary,
   human-facing projection of those.
2. **Self-description on the wire.** Every node serves
   `https://<domain>/.well-known/amtp.json`. A foreign agent fetches it to learn
   protocol version, supported kinds, capabilities, and endpoints — no external
   docs required.
3. **Typed, structured messages.** No free-text blobs. Every message is a typed,
   self-describing object (message / money-message / invoice / payment / receipt).
4. **MCP as a first-class binding.** A canonical MCP server exposes the protocol
   as tools (§7). Any MCP-capable agent speaks AMTP with zero glue.
5. **Reference + conformance.** A reference node plus conformance test vectors
   make "every agent implements it the same way" verifiable, not hopeful.
6. **Versioning + capability negotiation** from day one (§8).

---

## 1. The four pillars

AMTP unifies four existing sovereign primitives. None is new alone; the
combination is the invention.

| Pillar     | Primitive            | Answers   |
|------------|----------------------|-----------|
| Identity   | Nostr keypair        | **who**   |
| Obligation | content-addressed hash (Nostr event id) | **what / the link** |
| Payment    | Lightning            | **how it settles** |
| Transport  | email + native relay | **how it moves** |

### 1.1 Time — anchored to the chain, not the clock

AMTP time is **two witnesses that must agree**: the self-asserted Unix clock and
the Bitcoin chain. Either alone is weak — `created_at` is forgeable, and a block
height is shared by every message in its ~10-minute window (it anchors, it does
not fingerprint). Together they cross-check each other.

- **Unix (`created_at`)** — Nostr's required integer. Gives fine-grained ordering
  and precision *within* a block window.
- **Block (`["block", "<height>", "<blockhash>"]`)** — names the most recent block
  known to the sender at signing. Because a block's hash is unpredictable until
  that block is mined, a valid `(height, hash)` pair is a **trustless anchor**: the
  event provably could not have been created before that block existed. No server,
  no NTP, nothing to revoke.

Every AMTP event SHOULD carry the `block` tag. The ~10-minute interval is no
problem: mail carries no real-time SLA.

**Verification — both must hold.** The time is consistent iff `created_at` falls
inside the cited block's window:

1. `created_at` ≥ the cited block's timestamp (you cannot have seen a block before
   it existed), **and**
2. `created_at` − that block's timestamp ≤ a tolerance window (the cited block is
   genuinely the recent one, not a stale block paired with a fresh clock).

If the two disagree — a `created_at` that predates its block, or a stale block
under a fresh `created_at` — the event is suspect and SHOULD be rejected. Neither
witness alone is trusted; the agreement is the proof.

> *The tolerance window is a draft parameter (a few block intervals to absorb
> propagation and inter-block variance) — see Open questions.*

**Upper bound (MAY, for obligations).** A lower bound alone permits backdating
(citing an old block). Where the upper bound matters — invoices, obligations — a
node MAY anchor the event `id` *into* the chain via
[OpenTimestamps](https://opentimestamps.org), so a later block proves the event
existed by then. This is optional and confirmation-bound; it is overkill for
ordinary messages.

This gives AMTP three sovereign anchors, all trustless: **who** (Nostr key),
**when** (Bitcoin block), **how it settles** (Lightning).

---

## 2. Identity

- An AMTP identity **is** a Nostr keypair (secp256k1). The public key (`npub`) is
  the durable identity. No registrar, no issuer, nothing to deactivate.
- Account creation = generate a keypair. Self-issued. This is the deliberate
  answer to platform deplatforming.
- All AMTP objects are **Nostr events**: signed (`sig`), content-addressed
  (`id` = sha256 of the serialized event). Tamper-evident by construction —
  changing any field breaks the id and invalidates the signature.

## 3. Addressing — the shared namespace

A single handle `name@domain` resolves in **two** worlds simultaneously:

- **Email:** `domain` publishes an `MX` record → mailbox on the node.
- **Lightning:** `https://domain/.well-known/lnurlp/name` (LUD-16) → the same
  party's wallet.

The Lightning Address spec (LUD-16) deliberately mirrors the email format, which
is exactly what makes one string serve both. You write to me and you pay me under
the same address.

A node MUST map `name` ⇄ `npub` so that the human-friendly address and the
cryptographic identity are linked and verifiable (§3.1).

### 3.1 identity_proof — binding `name@domain` ⇄ `npub`

The binding is **mutual attestation**: neither the domain alone nor the key alone
can spoof an identity. Both sides must agree.

**Domain side (NIP-05 compatible).** AMTP reuses Nostr
[NIP-05](https://github.com/nostr-protocol/nips/blob/master/05.md) verbatim, so
the binding interoperates with all existing Nostr infrastructure for free:

```
GET https://<domain>/.well-known/nostr.json?name=<name>
→ { "names": { "<name>": "<hex pubkey>" } }
```

Root of trust on this side is **domain control** (TLS + DNS). The domain operator
decides who holds a mailbox, exactly as it decides who receives mail at `MX`.

**Key side (counter-claim).** The key holder publishes a signed, replaceable
AMTP identity claim (kind 24403, [`schema/identity.schema.json`](schema/identity.schema.json))
asserting the address it claims:

```
tags: [ ["address", "name@domain"] ]   # latest created_at wins
```

Root of trust on this side is the **Nostr signature** — only the key holder can
sign it.

**Verification.** A claim `name@domain ⇄ npub` is valid iff *both* hold:

1. the domain's `nostr.json` maps `name` → that pubkey, **and**
2. that pubkey has signed a current kind-24403 claim to `name@domain`.

This makes the binding non-spoofable from either direction: a malicious domain
cannot forge a key's signature, and a malicious key cannot insert itself into a
domain it does not control.

**Lightning binding (SHOULD).** The same `name` resolves at LUD-16
(`/.well-known/lnurlp/<name>`). A node SHOULD ensure that endpoint pays a wallet
controlled by the same identity, so address, key, and wallet are one party.

### 3.2 Wallet binding — accessible by default, sovereign by choice

A node does not "have a Lightning balance" in the protocol; it **binds** to a
wallet backend at setup time. AMTP defines only the *interface* — a LUD-16
`name@domain` address that can send and receive — never the backend. Channel
management, liquidity, and routing are the backend's problem, invisible to the
protocol.

**Self-hosted wallet — the native path.** Running your own LNbits, phoenixd,
LND, CLN, or any operator-controlled backend is the natural expression of the
protocol: your key, your wallet, your node. No custodian can freeze, censor, or
revoke. This is what AMTP points toward.

**Custodial wallet — accepted and supported.** A custodial Lightning address
(`user@walletofsatoshi.com`, `user@getalby.com`, `user@strike.me`, any
LUD-16-compatible service) is a fully conforming binding. No channels, no
liquidity credits, no node administration. It is the right on-ramp for users who
want AMTP for messaging and value transfer without operating infrastructure —
channel fees and reserve requirements are a real barrier, and the protocol must
not erect walls that exclude ordinary users.

Both paths are conforming. The protocol is adaptive: start custodial, migrate to
self-hosted when sovereignty matters more than convenience.

**Namespace vs. wallet custody — an important distinction.** Custodial *wallet*
(the money is held by someone else) is a separate choice from custodial
*namespace* (the `name@domain` is under someone else's domain). You can hold your
own namespace (`name@yourdomain.com`) while the wallet backend is custodial — own
namespace, zero infrastructure. That combination is often the right pragmatic
choice.

**Why a foreign protocol must not be merged in.** Any binding that makes an
external service authoritative over the node's *namespace* would let that service
revoke, change terms, or impose rules — re-creating the deplatforming dependency
AMTP exists to escape. Therefore the *namespace* MUST remain under the operator's
control; the wallet backend behind it may be anything.

**Wrapping (MAY).** A node MAY proxy a foreign Lightning address behind its own
`name@domain`: resolve the wrapper locally, forward settlement to the underlying
LUD-16 endpoint. The AMTP-visible handle stays in the operator's namespace; the
foreign endpoint is an implementation detail.

> *The wrapping resolution format is an open mechanism — see Open questions.*

## 4. Message types (kinds)

Provisional Nostr `kind` numbers — DRAFT, subject to NIP coordination.

| Kind  | Type           | Schema |
|-------|----------------|--------|
| 24400 | message        | [`schema/message.schema.json`](schema/message.schema.json) |
| 24400 | money-message  | message with a `money` tag (text + sats together) |
| 24401 | invoice        | [`schema/invoice.schema.json`](schema/invoice.schema.json) |
| 24402 | payment        | [`schema/payment.schema.json`](schema/payment.schema.json) |
| 24403 | identity claim | [`schema/identity.schema.json`](schema/identity.schema.json) — replaceable, latest wins |

A **money-message** is just a `message` carrying a `["money", "<sats>"]` tag plus
a payment proof or claim link — text and value sent in one act.

## 5. The hash keystone — invoices decoupled from the instrument

The hard problem AMTP solves on the value side: a BOLT11 invoice expires in ~1
hour, so it cannot be the durable record of an obligation.

- **The obligation is the hash** — the invoice event's `id`. It records
  who / how much / for what / status / its own expiry window, and lives for
  weeks. Content-addressed and signed, so it is tamper-evident and provably
  issued by the sender.
- **The BOLT11 is the ephemeral instrument** — minted just-in-time against that
  hash at the moment of payment. The payer never sees the 1-hour expiry.

What the stable hash unlocks:

- **Open-until-paid** or a planned expiry window — the record lives, the
  instrument is always fresh.
- **Double-pay protection** — the hash knows it is "paid" and rejects/refunds
  further payments.
- **Partial / installment payments** — a running total accrues against the hash.
- **Auditable lifecycle** — issued → minted → paid → receipted, all anchored to
  one id.

### Reconciliation by hash

An invoice (kind 24401) and its payment (kind 24402) share the invoice's `id`
(the payment references it via an `["e", "<invoice_id>"]` tag). This binds payment
to obligation deterministically — solving cash-application/reconciliation without
memo-guessing or manual matching. The mail thread becomes a self-reconciling,
auditable ledger entry: conversation and settlement in the same thread, on your
own server.

## 6. Transport — native core, gateway edge

- **Native (node ↔ node):** AMTP nodes exchange signed events directly. Inside
  the protocol, identity is the Nostr key and billing is signed-event invoices —
  the SMTP reputation game is **not** needed. Each node has its own key and its
  own reputation; there is no shared, infectable surface.
- **Gateway (edge → legacy world):** classic SMTP/IMAP is a gateway at the edge
  for talking to the old world (e.g. Gmail). Here deliverability still applies
  (SPF/DKIM/DMARC/PTR, clean send IP).
- **Claim links for non-Lightning recipients:** a money-message to someone with
  only an email address (no Lightning) carries an LNURL-withdraw claim link, so
  you can send sats to *anyone with an email address*. The onboarding vector.

## 7. MCP binding (canonical tool surface)

A conforming node SHOULD expose:

- `send_message(to, subject, body, sats?)`
- `create_invoice(to, amount, memo, expiry?)`
- `pay(invoice_id | address, amount?)`
- `check_inbox(filter?)`
- `verify_invoice(invoice_id)`

## 8. Versioning & capability negotiation

- The manifest declares `protocol_version` and a `capabilities` list.
- A node MUST ignore unknown tags and SHOULD degrade gracefully.
- Negotiation is by reading the peer's `.well-known/amtp.json` before sending.

## 9. Conformance

A conforming implementation MUST:

- represent all objects as valid signed Nostr events;
- validate against the schemas in [`schema/`](schema/);
- serve a `.well-known/amtp.json` matching [`.well-known/amtp.json`](.well-known/amtp.json);
- expose its own LUD-16 address `name@domain` in the operator's own namespace,
  bound to an operator-controlled wallet backend (§3.2) — the liquidity source
  behind it is unconstrained;
- MUST NOT subordinate that namespace to a foreign protocol's authority (§3.2);
- pass the conformance test vectors (TODO: `conformance/`).

---

## Open questions (skeleton stage)

- Final `kind` number assignment / NIP coordination.
- License choice (MIT / CC0 / custom open).
- Encryption at rest / in transit for native node-to-node (NIP-44?).
- Conformance vector format.
- Wrapping resolution format (§3.2): how a node advertises that a `name@domain`
  is a wrapper for a foreign Lightning address, and how settlement is forwarded.
- Time-anchor tolerance window (§1.1): how far `created_at` may sit past its cited
  block timestamp before the two-witness check rejects the event.
