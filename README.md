# AMTP — Agentic Mail Transfer Protocol

**A mail protocol where one handle is both an address and a wallet.** Identity is
a key nobody issued and nobody can revoke. Messages can carry value.

Self-hostable by design — your own LNbits or Lightning node is the natural home.
Custodial Lightning addresses are also fully supported for anyone who wants to
start without running infrastructure. Open, free, sovereignty is a spectrum.

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

**Native path — self-hosted node**: run your own Lightning backend (LNbits, LND,
CLN, phoenixd, …) and serve `/.well-known/lnurlp/<name>` from your own domain.
Your key, your wallet, your node. This is what the protocol is built for.

**Accepted path — custodial Lightning address**: if you already have
`you@walletofsatoshi.com`, `you@getalby.com`, or any LUD-16 address, that *is*
a valid AMTP binding. No channels, no liquidity credits, zero infrastructure.

Start where you are. Migrate toward sovereignty when it matters.

## Lineage

Sibling to the Universal Memory Protocol and CAIP — same Satoshi-pattern: don't
build a company, build a protocol and release it.

---

*Why we build this: [`GENESIS.md`](GENESIS.md).*
