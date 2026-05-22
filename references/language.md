# DAML Language Reference

DAML is a strongly-typed, pure functional language (Haskell-derived) specialized
for ledger contracts. This file covers the language; for testing see
[daml-script.md](daml-script.md), for tooling see [dpm-cli.md](dpm-cli.md).

## Modules and imports

```daml
module Main where            -- module name must match the file path under `source:`

import DA.List               -- standard library
import DA.Optional (fromOptional, isSome)
import DA.Text qualified as T -- qualified import; use as T.length
import Daml.Script            -- only in test modules
```

A file at `daml/Asset/Iou.daml` must declare `module Asset.Iou where`.

## Templates

A `template` declares a contract type. An instance ("a contract") is an immutable
record on the ledger plus the parties authorized over it.

```daml
template Account
  with
    bank    : Party
    owner   : Party
    balance : Decimal
    note    : Optional Text
  where
    signatory bank, owner          -- BOTH must authorize create/archive
    observer  []                   -- additional read-only parties (optional)
    ensure balance >= 0.0          -- invariant checked on every create

    -- optional contract key: unique lookup value
    key (bank, owner) : (Party, Party)
    maintainer key._1              -- maintainers must be a subset of signatories
```

| Clause       | Meaning |
|--------------|---------|
| `with`       | Contract fields (the record payload). |
| `signatory`  | Parties who authorize creation/archival and see all activity. ≥ 1 required. |
| `observer`   | Parties who can see the contract but grant no authority. |
| `ensure`     | Boolean invariant; a `create` that violates it is rejected. |
| `key` / `maintainer` | Optional unique key; maintainers ⊆ signatories. |
| `choice`     | Named action exercisable on the contract (see below). |

`this` refers to the current contract's record. `create this with field = v`
makes an updated copy — the idiomatic way to "modify" an immutable contract.

## Choices

A choice is the only way to act on a contract. Exercising it runs an `Update`.

```daml
    choice Deposit : ContractId Account     -- return type after the colon
      with
        amount : Decimal                    -- choice arguments (a record)
      controller owner                      -- party authorizing the exercise
      do
        assertMsg "amount must be positive" (amount > 0.0)
        create this with balance = balance + amount
```

### Consumption modifiers

| Modifier        | Behavior |
|-----------------|----------|
| (default)       | **Consuming** — archives the contract *before* the body runs. |
| `nonconsuming`  | Contract survives; use for read-only or repeatable actions. |
| `preconsuming`  | Archives *before* the body — body cannot `fetch` the contract. |
| `postconsuming` | Archives *after* the body — body can still use the contract. |

A consuming choice archives `this`, so return the `ContractId` of any successor
contract. Callers using the old id afterward get a "contract not active" error.

### Controllers and authorization

```daml
      controller owner                 -- single controller
      controller bank, owner           -- multiple: all must authorize
```

The flexible (modern) controller syntax also allows controllers computed from
choice arguments. A choice runs only if every controller is an authorizing party
of the submitting transaction.

### `Update` actions (inside choice/`do` blocks)

| Action | Purpose |
|--------|---------|
| `create c` | Create a contract, returns `ContractId`. |
| `exercise cid Ch with ...` | Exercise a choice on a contract. |
| `exerciseByKey @T key Ch with ...` | Exercise by contract key. |
| `fetch cid` | Read a contract's payload (requires authorization/visibility). |
| `fetchByKey @T key` | Fetch by key, returns `(ContractId, payload)`. |
| `lookupByKey @T key` | `Optional (ContractId T)` — no failure if absent. |
| `archive cid` | Shorthand for `exercise cid Archive`. |
| `getTime` | Ledger effective time (`Time`). Never use a wall clock. |
| `assertMsg msg cond` / `abort msg` | Fail the transaction with a message. |

Use `create`/`exercise` inside `Update` (choice bodies). Use the `*Cmd` variants
(`createCmd`, `exerciseCmd`) inside Daml `Script`.

## Data types

```daml
-- Record
data Address = Address with
    street : Text
    city   : Text
  deriving (Eq, Show, Ord)

-- Variant (sum type) with payloads
data Instruction
  = Pay with to : Party; amount : Decimal
  | Cancel
  deriving (Eq, Show)

-- Enum
data Color = Red | Green | Blue deriving (Eq, Show, Enum, Bounded)

-- Type synonym
type Amount = Decimal
```

### Built-in types

| Type | Notes |
|------|-------|
| `Int` | 64-bit signed integer. |
| `Decimal` | Fixed-point `Numeric 10`. Use for money — there is **no `Float`**. |
| `Numeric n` | Fixed-point with `n` decimal places. |
| `Text` | Unicode string (use `DA.Text`). |
| `Bool` | `True` / `False`. |
| `Party` | A ledger participant identity. |
| `Date`, `Time` | Calendar date / UTC timestamp (`DA.Date`, `DA.Time`). |
| `ContractId a` | Reference to an active contract of template `a`. |
| `Optional a` | `Some a` or `None`. |
| `[a]` | List. |
| `(a, b)` | Tuple; access with `._1`, `._2`. |
| `Map k v`, `Set a` | `DA.Map`, `DA.Set`. `TextMap` keyed by `Text`. |
| `Update a`, `Script a` | Effect types for ledger actions / test scripts. |

## Functions and functional style

```daml
-- Top-level function with a type signature (always write one)
transfer : Party -> Decimal -> Update (ContractId Account)
transfer newOwner amt = ...

-- let / where
discounted : Decimal -> Decimal
discounted x =
  let rate = 0.9 in x * rate

afterFee : Decimal -> Decimal
afterFee x = x - fee
  where fee = x * 0.01

-- lambda
double = \x -> x * 2

-- pattern matching
describe : Optional Int -> Text
describe o = case o of
  None   -> "nothing"
  Some 0 -> "zero"
  Some n -> "got " <> show n

-- if/then/else is an expression
sign x = if x >= 0 then "pos" else "neg"
```

`do` notation sequences `Update` (choice bodies) and `Script` (tests). `pure x`
(or `return x`) lifts a value; `<-` binds a result; `let` binds pure values.

```daml
do
  cid <- create someContract        -- bind an Update result
  let label = "done"                -- pure binding, no `in`
  pure (cid, label)
```

## Type classes

```daml
class Describable a where
  describe : a -> Text

instance Describable Color where
  describe Red = "red"
  describe Green = "green"
  describe Blue = "blue"
```

Common derivable classes: `Eq`, `Ord`, `Show`, `Enum`, `Bounded`. Add them with
`deriving (...)` on a `data` declaration.

## Interfaces (polymorphism across templates)

Interfaces let unrelated templates expose a shared set of choices/views.

```daml
interface IAsset where
  viewtype AssetView
  getOwner : Party
  choice Reassign : ContractId IAsset
    with newOwner : Party
    controller getOwner
    do pure (toInterfaceContractId @IAsset self)

data AssetView = AssetView with owner : Party

template Iou
  with owner : Party; amount : Decimal
  where
    signatory owner
    interface instance IAsset for Iou where
      view = AssetView with owner
      getOwner = owner
```

Use `toInterface` / `fromInterface` and `toInterfaceContractId` to convert
between a concrete template and an interface.

## Common standard-library modules

| Module | Provides |
|--------|----------|
| `DA.List` | `map`, `filter`, `foldl`, `sortOn`, `find`, `head`, `dedup`. |
| `DA.Optional` | `fromOptional`, `isSome`, `optional`, `mapOptional`, `catOptionals`. |
| `DA.Text` | Text ops: `length`, `splitOn`, `isPrefixOf`, `intercalate`. |
| `DA.Map` / `DA.Set` | Ordered map/set operations. |
| `DA.Foldable` / `DA.Traversable` | `forA_`, `mapA`, `sequence`. |
| `DA.Action` | `when`, `unless`, `void`. |
| `DA.Assert` | `assertMsg`, `(===)`, `(=/=)` — used heavily in tests. |
| `DA.Date` / `DA.Time` | Date/time construction and arithmetic. |
| `Daml.Script` | Test/scripting API (test modules only). |

## Idioms and gotchas

- Always write top-level type signatures — they document intent and sharpen
  compiler errors.
- Prefer `assertMsg` (message included) over bare `assert`.
- Don't recompute authorization in a choice body; the `controller` clause is the
  authorization. Use `ensure` for data invariants.
- `fetch` requires the fetching party to be a stakeholder of the target contract —
  a `fetch` that "should work" but fails is usually a visibility problem.
- Comments: `--` line, `{- ... -}` block, `-- |` doc comment (picked up by
  `dpm docs`).
