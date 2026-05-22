# Testing with Daml Script

Daml Script is the primary tool for testing DAML contracts. A script submits
commands and queries as multiple parties against a **fresh, empty in-memory
ledger** (the "IDE ledger"). Tests are deterministic and fast — they are the
bottom of the testing pyramid.

Run scripts two ways:
- **CLI / CI:** `dpm test` runs every `Script` in the package and prints
  pass/fail plus template/choice **coverage**.
- **VS Code:** click the **"Script results"** code lens above a script to inspect
  the table view (active contracts) and transaction view (who saw what).

## Anatomy of a test

```daml
module Test.Iou where

import Daml.Script
import DA.Assert ((===), (=/=))
import Main                       -- the module under test

-- A test is a top-level value of type `Script a`.
-- Name tests `test_*` (or a clear descriptive name) so intent is obvious.
test_transfer : Script ()
test_transfer = script do
  -- 1. Arrange: allocate the parties.
  issuer <- allocateParty "Issuer"
  alice  <- allocateParty "Alice"
  bob    <- allocateParty "Bob"

  -- 2. Act: submit commands as a party.
  iou <- submit issuer do
    createCmd Iou with issuer; owner = alice; amount = 100.0; currency = "USD"

  iou' <- submit alice do
    exerciseCmd iou Transfer with newOwner = bob

  -- 3. Assert (happy path).
  Some c <- queryContractId bob iou'
  c.owner === bob

  -- 4. Assert (failure path) — never skip this.
  submitMustFail alice do
    exerciseCmd iou' Transfer with newOwner = issuer   -- alice no longer owns it
```

The `script do` wrapper is the conventional form. A bare `do` of type
`Script ()` also works.

## Parties

| Function | Purpose |
|----------|---------|
| `allocateParty "Alice"` | Allocate a fresh party with a display-name hint. |
| `allocatePartyWithHint "Alice" (PartyIdHint "alice")` | Allocate with a stable id hint. |
| `listKnownParties` | List parties known to the ledger. |

## Submitting commands

Commands run as one or more parties; inside a `submit` block use the **`*Cmd`**
builders, not the bare `Update` actions.

| Builder | Purpose |
|---------|---------|
| `createCmd t` | Create a contract. |
| `exerciseCmd cid Ch with ...` | Exercise a choice. |
| `exerciseByKeyCmd @T key Ch with ...` | Exercise by contract key. |
| `createAndExerciseCmd t Ch with ...` | Create then immediately exercise, atomically. |
| `archiveCmd cid` | Archive a contract. |

| Submitter | Purpose |
|-----------|---------|
| `submit party cmds` | Run as a single party; fails the test if the ledger rejects. |
| `submitMustFail party cmds` | Asserts the submission **is rejected**. |
| `submitMulti [actAs] [readAs] cmds` | Multiple acting parties + read-only parties. |
| `submitTree party cmds` | Submit and return the full transaction tree (for assertions). |

`submitMustFail` is how you prove authorization rules, `ensure` invariants, and
`assertMsg` guards actually hold. A test suite with only happy paths is incomplete.

## Querying ledger state

| Query | Returns |
|-------|---------|
| `query @T party` | `[(ContractId T, T)]` — all visible `T` contracts for `party`. |
| `queryContractId party cid` | `Optional T` — payload if `party` can see it. |
| `queryContractKey @T party key` | `Optional (ContractId T, T)` — lookup by key. |
| `queryFilter @T party predicate` | Visible `T` contracts matching a predicate. |

Visibility matters: `query` only returns contracts the queried party is a
stakeholder of. Querying as the wrong party is a common reason a test "loses" a
contract.

## Assertions

```daml
import DA.Assert ((===), (=/=))

c.owner === bob                       -- equality assert with a good failure message
c.amount =/= 0.0                      -- inequality assert
assertMsg "balance must be positive" (c.balance > 0.0)
assert (c.balance > 0.0)              -- prefer assertMsg — include a message
```

Pattern-matching a query result also asserts shape:

```daml
Some c <- queryContractId bob iou'    -- fails the script if the result is None
[(cid, acct)] <- query @Account bank  -- fails unless exactly one contract exists
```

## Controlling time

The IDE ledger starts at a fixed epoch. Control it explicitly — never rely on
wall-clock time.

| Function | Effect |
|----------|--------|
| `getTime` | Current ledger time. |
| `setTime t` | Set ledger time to an absolute `Time`. |
| `passTime (days 7)` | Advance ledger time by a `RelTime`. |
| `sleep (seconds 1)` | Pause real execution (rarely needed in unit tests). |

```daml
import DA.Time (days)
passTime (days 30)
now <- getTime
```

## Debugging

| Tool | Use |
|------|-----|
| `debug : Text -> m ()` | Print inside a `Script`/`Update` action. Output tags the source line, e.g. `[Test.Iou:42]`. |
| `trace : Text -> a -> a` | Print when a pure expression is evaluated. |
| VS Code transaction view | Inspect which party saw each create/exercise — best tool for visibility bugs. |

## Multi-party workflow example

```daml
test_escrow : Script ()
test_escrow = script do
  seller <- allocateParty "Seller"
  buyer  <- allocateParty "Buyer"
  agent  <- allocateParty "Agent"

  proposal <- submit seller do
    createCmd EscrowProposal with seller; buyer; agent; price = 500.0

  -- a two-party agreement: submitMulti with both acting parties
  escrow <- submitMulti [buyer, agent] [] do
    exerciseCmd proposal Accept

  -- agent settles; assert the resulting state
  submit agent do exerciseCmd escrow Settle
  remaining <- query @Escrow agent
  remaining === []
```

## Organizing tests

- Put scripts in a `Test/` module subtree (`daml/Test/Iou.daml` →
  `module Test.Iou`). This keeps test-only code and the `daml-script` dependency
  off the production contract surface.
- One script per scenario; name them for the behavior under test.
- For larger suites, factor shared setup into helper functions returning
  `Script` values and reuse them across tests.
- `dpm test` reports choice/template **coverage** — drive every choice at least
  once, including its rejection paths.

## Beyond unit tests

Daml Script gives fast, deterministic logic tests. For integration testing
against real Ledger API semantics, multi-participant topologies, and
authentication, run against `dpm sandbox` (see [dpm-cli.md](dpm-cli.md)). The
same `Script` code can target a live ledger via `dpm script --ledger-host ...`.
