# PartyLayer — Canton wallet integration

PartyLayer is one option for connecting a browser dApp to a Canton Network
wallet. It is a TypeScript SDK that wraps multiple wallets behind a single API
so the dApp does not have to ship an adapter per wallet. Use it when the dApp
needs to support several wallets, or when you want a typed, CIP-0103-compliant
session abstraction instead of talking to `window.canton.*` directly.

- Repo: <https://github.com/PartyLayer/PartyLayer>
- Standard: CIP-0103 (Canton dApp ↔ wallet communication)
- License / status: check the repo — this skill does not vendor it.

## What it gives you

- **Unified API across wallets.** Built-in adapters for Console Wallet, 5N Loop,
  Cantor8, Nightly, and Bron. Any CIP-0103 wallet that exposes itself on
  `window.canton.*` is auto-discovered at runtime — no extra configuration.
- **Typed sessions.** A successful connect returns a session whose `partyId`
  identifies the connected Canton party. Treat that string as the DAML `Party`
  your dApp will submit commands as (or query on behalf of) through the JSON
  Ledger API.
- **Session management.** Encrypted, origin-bound sessions so a session granted
  to one dApp origin cannot be replayed by another.
- **React wrapper.** `PartyLayerKit` (zero-config) creates the client and
  registers the built-in adapters; the vanilla JS client is available for
  non-React apps.

## Install

```sh
npm install @partylayer/sdk @partylayer/react
```

Requires Node 18+ and (for the React wrapper) React 18+. Next.js is supported.

## Where it fits in a Canton dApp

```
┌────────────────────┐    CIP-0103     ┌────────────────────┐
│  Browser dApp      │ ──────────────▶ │  Wallet (any of    │
│  (React / TS)      │                 │  Console, 5N Loop, │
│                    │                 │  Cantor8, Nightly, │
│  @partylayer/sdk   │                 │  Bron, …)          │
│  @partylayer/react │                 └─────────┬──────────┘
└─────────┬──────────┘                           │ signs commands
          │ session.partyId                      ▼
          │                              ┌──────────────────┐
          └─────────── JSON Ledger ─────▶│  Canton ledger   │
                       API (separate)    └──────────────────┘
```

PartyLayer covers the **wallet ↔ dApp** edge only. Submitting commands to the
ledger (JSON Ledger API, gRPC Ledger API, or codegen bindings from
`dpm codegen`) is a separate concern and unaffected by which wallet abstraction
you choose.

## When *not* to use it

- **Backend / service signer.** If a service holds its own party keys (a daemon,
  cron job, or trusted automation), use the JSON Ledger API + JWT directly. A
  browser wallet abstraction is the wrong shape.
- **You only ever target one wallet** and that wallet's native API is already
  comfortable. The abstraction adds dependencies you do not need.
- **Pure off-ledger UI.** If the dApp never submits commands itself, you do not
  need a wallet at all.

## Verifying claims

This page summarizes the project's README; for current adapter list, exact API
surface, session payload shape, and version compatibility, read the repo
directly — APIs evolve and this file is not regenerated automatically.
