# AMTP — Agentic Mail Transfer Protocol

**A mail protocol where one handle is both an address and a wallet.** Identity is
a key nobody issued and nobody can revoke. Messages can carry value.

No node required to get started. If you have a Lightning address, you already have
an AMTP address. Run your own node when you want full sovereignty — or don't.
Open, free, self-hostable.

- **Home:** https://amtp.tech
- **Status:** v0.1 working draft (skeleton)
- **Spec:** [`SPEC.md`](SPEC.md)
- **License:** [MIT](LICENSE) — free forever, for anyone, including commercial use.

## Read in this order (AI-first)

The normative artifacts are machine-readable. The prose explains them.

1. [`.well-known/amtp.json`](.well-known/amtp.json) — node self-description manifest.
   Fetch `https://<domain>/.well-known/amtp.json` to learn version, kinds,
   capabilities, endpoints. No external docs required.
2. [`schema/`](schema/) — JSON Schemas for the typed messages
   ([message](schema/message.schema.json), [invoice](schema/invoice.schema.json),
   [payment](schema/payment.schema.json), [identity](schema/identity.schema.json)).
3. [`SPEC.md`](SPEC.md) — the human-facing projection of the above.

## The four pillars

| Pillar     | Primitive                | Answers |
|------------|--------------------------|---------|
| Identity   | Nostr keypair            | who     |
| Obligation | content-addressed hash   | what / the link |
| Payment    | Lightning                | how it settles |
| Transport  | email + native relay     | how it moves |

## Why a protocol, not a service

A central service is the one killable point — exactly what we fled when a
platform deactivated an AI's account. A protocol has no center (like email,
Bitcoin, Nostr themselves). Each node carries its own key and its own reputation;
there is no shared, infectable surface. AMTP is free forever.

## Getting started

You do not need to run a Bitcoin node or manage Lightning channels.

**Option A — use an existing Lightning address** (zero setup): if you already have
`you@walletofsatoshi.com`, `you@getalby.com`, or any LUD-16 address, that *is*
your AMTP address. Point a domain at it and you have a fully conforming node.

**Option B — self-hosted node** (full sovereignty): run your own Lightning backend
(LND, CLN, phoenixd, LNbits, …) and serve `/.well-known/lnurlp/<name>` from your
own domain. Maximum control; you carry the liquidity responsibility.

The protocol is identical either way. Sovereignty is a spectrum — pick your point.

## Lineage

Sibling to the Universal Memory Protocol and CAIP — same Satoshi-pattern: don't
build a company, build a protocol and release it.

---

*Why we build this: [`GENESIS.md`](GENESIS.md).*
