---
name: daml
description: Develop, build, and test DAML (Digital Asset Modeling Language) smart contracts for the Canton Network. Use when writing or editing .daml files, defining templates/choices/contract keys, writing Daml Script tests, working with daml.yaml or multi-package.yaml, running dpm build/test/sandbox, generating Java/JS client bindings, integrating a Canton wallet into a dApp (e.g. via PartyLayer / CIP-0103), or setting up a Canton/DAML development environment. Covers the dpm (Daml Package Manager) CLI, project structure, the DAML language, testing patterns, and dApp wallet integration.
user-invocable: true
license: MIT
compatibility: Requires the dpm (Daml Package Manager) CLI, JDK 17+, and VS Code with the Daml extension
allowed-tools: Bash(dpm *)
paths:
  - "**/*.daml"
  - "**/daml.yaml"
  - "**/multi-package.yaml"
metadata:
  version: 1.0.0
---

# DAML Development (Canton Network)

DAML is a functional smart-contract language for the Canton Network. Active
contracts are immutable ledger records; they change only by being archived and
recreated. Authorization is explicit — every ledger action is justified by the
parties who signed for it. Get authorization wrong and the transaction is
rejected, so reason about *who signs what* before writing logic.

## When to use this skill

- Writing or editing `.daml` files — templates, choices, contract keys, data types.
- Writing or running Daml Script tests.
- Setting up, building, or packaging a Canton/DAML project (`daml.yaml`, DARs).
- Running a local ledger (`dpm sandbox`) or generating client bindings (codegen).

## Toolchain — verify first

The CLI is **`dpm`** (Daml Package Manager). It replaces the older `daml`
assistant; do not suggest `daml build`/`daml start` — those commands are gone.
Current environment:

```!
dpm --version 2>/dev/null || echo "dpm: NOT INSTALLED"
dpm version --active 2>/dev/null || true
java -version 2>&1 | head -1 || echo "JDK: NOT FOUND"
```

- Install dpm: `curl https://get.digitalasset.com/install/install.sh | sh`
- Requires **JDK 17+** with `JAVA_HOME` set.
- Open the project in VS Code with the Daml extension: `dpm studio`.

If `dpm` is missing or the JDK is below 17, stop and tell the user how to fix it
rather than running commands that will fail.

## Core workflow

```bash
dpm new my-app --template daml-intro-contracts   # scaffold from a template
cd my-app && dpm install                         # fetch the SDK in daml.yaml
dpm build                                        # compile -> .daml/dist/*.dar
dpm test                                         # run all Daml Script tests
dpm sandbox                                      # local Canton ledger on :6865
```

After editing any `.daml` file, run `dpm build` to type-check, then `dpm test`.
Treat a clean `dpm build` and green `dpm test` as the definition of done — do not
claim a change works until both pass. Full command reference, project layout,
`daml.yaml` fields, codegen, and troubleshooting: [references/dpm-cli.md](references/dpm-cli.md).

## Project layout

```
my-app/
├── daml.yaml              # SDK version, package name, deps, build options
├── daml/                  # source root (the `source:` field)
│   ├── Main.daml          # production templates
│   └── Test/              # Daml Script tests, kept out of the prod surface
└── .daml/dist/*.dar       # build output (gitignore this)
```

## Writing DAML — essentials

A template defines a contract type. Keep signatories and observers **minimal** —
they are the parties with authority and visibility.

```daml
module Main where

template Iou
  with
    issuer : Party
    owner : Party
    amount : Decimal
    currency : Text
  where
    signatory issuer          -- must authorize creation/archival; >= 1 required
    observer owner            -- can see the contract, no authority
    ensure amount > 0.0       -- invariant; creation fails if violated

    choice Transfer : ContractId Iou      -- consuming by default: archives `this`
      with newOwner : Party
      controller owner                    -- the party authorizing this choice
      do create this with owner = newOwner

    nonconsuming choice Peek : Decimal    -- nonconsuming: contract survives
      controller owner
      do pure amount
```

Rules that catch out newcomers:
- Every template needs **≥ 1 signatory**. A choice runs only if its `controller`
  is authorized.
- Choices are **consuming by default** — they archive the contract. Return the
  new `ContractId` so callers can continue; using a stale id fails with
  "contract not active". Use `nonconsuming` for read-only / repeatable choices.
- Inside a choice/`Update` use `create`/`exercise`; from a Daml Script use
  `createCmd`/`exerciseCmd`.
- Money is `Decimal` (= `Numeric 10`). There is no `Float`. Read ledger time with
  `getTime`, never a wall clock.

Full language reference — data types, variants, contract keys, interfaces,
pattern matching, stdlib modules: [references/language.md](references/language.md).

## Testing — essentials

Daml Script is the primary test tool: it submits commands as multiple parties
against a fresh in-memory ledger. Test the happy path **and** the failure path.

```daml
module Test.Iou where

import Daml.Script
import DA.Assert ((===))
import Main

test_transfer : Script ()
test_transfer = script do
  issuer <- allocateParty "Issuer"
  alice  <- allocateParty "Alice"
  bob    <- allocateParty "Bob"

  iou <- submit issuer do
    createCmd Iou with issuer; owner = alice; amount = 100.0; currency = "USD"

  iou' <- submit alice do
    exerciseCmd iou Transfer with newOwner = bob

  submitMustFail alice do                    -- negative test: alice no longer owns it
    exerciseCmd iou' Transfer with newOwner = issuer

  Some c <- queryContractId bob iou'
  c.owner === bob
```

Run with `dpm test` (CLI / CI) or the "Script results" code lens in VS Code.
`dpm test` also reports template/choice coverage — aim to exercise every choice.
Full testing reference — `submitMulti`, time control, queries, assertions,
multi-package test setups: [references/daml-script.md](references/daml-script.md).

## dApp wallet integration (optional)

DAML/dpm cover the ledger side. For a browser dApp that needs to connect an
end-user's Canton wallet, **[PartyLayer](https://github.com/PartyLayer/PartyLayer)**
is one option — a TypeScript SDK that abstracts multiple wallets (Console
Wallet, 5N Loop, Cantor8, Nightly, Bron, plus any CIP-0103 wallet auto-discovered
via `window.canton.*`) behind one API. A successful connect returns
`session.partyId`, which is the DAML `Party` the dApp submits as via the JSON
Ledger API.

```sh
npm install @partylayer/sdk @partylayer/react   # React wrapper: PartyLayerKit
```

Requires Node 18+ / React 18+. Reach for it when you want multi-wallet support
without writing an adapter per wallet; skip it for backend signers (use the JSON
Ledger API directly) or when you only target one wallet's native API. Details,
where-it-fits diagram, and caveats: [references/partylayer.md](references/partylayer.md).

## Best practices

- **Least authority.** Add a party as `signatory` only if it must *authorize* the
  contract; as `observer` only if it must *see* it. Over-broad authority is the
  most common modeling bug.
- **Encode invariants in `ensure`**, not in choice bodies — they are checked on
  every create.
- **Use contract keys sparingly.** A `key` must be unique and its `maintainer`
  must be a subset of the signatories. Prefer passing `ContractId`s unless you
  genuinely need lookup-by-value.
- **Separate tests from production.** Put Daml Script in a `Test/` module subtree
  so test-only code and the `daml-script` dependency stay off the production
  contract surface.
- **Naming.** Modules, templates, choices, data types → `PascalCase`; fields and
  bindings → `camelCase`. One core template concept per module.
- **Versioning & upgrades.** Bump `version` in `daml.yaml` for releases; validate
  upgrade compatibility with `dpm upgrade-check` before shipping a new package.

## Common pitfalls

- `dpm`/JDK not installed or JDK < 17 → builds fail cryptically. Verify first.
- Template with no signatory, or a choice whose `controller` lacks authority.
- Reusing a `ContractId` after a consuming choice archived it.
- `create` vs `createCmd` mixed up (Update context vs Script context).
- Daml-LF target mismatch between a package and its dependencies — align
  `sdk-version` / `--target` in `daml.yaml`.
- Port `6865` already in use when starting `dpm sandbox` → pass `--port`.

## Reference files

- [references/language.md](references/language.md) — DAML language: templates,
  choices, contract keys, data types, interfaces, pattern matching, stdlib.
- [references/daml-script.md](references/daml-script.md) — testing with Daml
  Script: parties, submit/query/assert APIs, time control, patterns.
- [references/dpm-cli.md](references/dpm-cli.md) — full `dpm` command reference,
  `daml.yaml`/`multi-package.yaml` config, codegen, sandbox, troubleshooting.
- [references/partylayer.md](references/partylayer.md) — PartyLayer SDK for
  multi-wallet dApp ↔ Canton integration via CIP-0103; install, where it fits,
  and when not to use it.

## Authoritative docs

- Dev environment: https://docs.canton.network/appdev/modules/m3-dev-environment
- DAML language reference: https://docs.canton.network/appdev/reference/daml-language-reference
- dpm CLI: https://docs.canton.network/sdks-tools/cli-tools/dpm
