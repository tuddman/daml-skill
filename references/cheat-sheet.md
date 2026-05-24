# DAML Cheat Sheet

A condensed, syntax-first reference adapted from the official cheat sheet
at <https://docs.daml.com/cheat-sheet/>. Use this when you need to recall a
specific keyword, builtin, or call signature; use [language.md](language.md),
[daml-script.md](daml-script.md), and [dpm-cli.md](dpm-cli.md) for deeper
explanation.

> **CLI note.** The upstream cheat sheet still uses the legacy `daml`
> assistant. This skill uses **`dpm`** (Daml Package Manager). Map as needed:
> `daml new → dpm new`, `daml build → dpm build`, `daml start → dpm start`,
> `daml sandbox → dpm sandbox`, `daml test → dpm test`, `daml studio →
> dpm studio`, `daml ledger upload-dar → dpm ledger upload-dar`,
> `daml json-api → dpm json-api`, `daml script → dpm script`. The flags are
> the same. See [dpm-cli.md](dpm-cli.md) for the full mapping.

## Command-line tools

| Action                                            | Command                                                                            |
| ------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Install the assistant                             | `curl https://get.digitalasset.com/install/install.sh \| sh`                       |
| Create a new Daml project                         | `dpm new <myproject>`                                                              |
| Create a new Daml/React full-stack project        | `dpm new <myproject> --template create-daml-app`                                   |
| Start the IDE                                     | `dpm studio`                                                                       |
| Build the project                                 | `dpm build`                                                                        |
| Build + start sandbox + JSON API                  | `dpm start`                                                                        |
| Start the sandbox (wall-clock time)               | `dpm sandbox`                                                                      |
| Start the sandbox (static time)                   | `dpm sandbox --static-time`                                                        |
| Start the JSON-API server                         | `dpm json-api --ledger-host localhost --ledger-port 6865 --http-port 7575`         |
| Upload a DAR to the ledger                        | `dpm ledger upload-dar <dar file>`                                                 |
| Run all test scripts + coverage                   | `dpm test --show-coverage --all --files Test.daml`                                 |
| Project configuration file                        | `daml.yaml`                                                                        |

## Comments & module header

```daml
let i = 1     -- end-of-line comment
{- delimited comment, can span lines -}

module Foo where
```

## Types

| Concept                | Syntax                                      |
| ---------------------- | ------------------------------------------- |
| Type annotation        | `myVar : TypeName`                          |
| Built-in types         | `Int, Decimal, Numeric n, Text, Bool, Party, Date, Time, RelTime` |
| Type synonym           | `type MyInt = Int`                          |
| Lists                  | `type ListOfInts = [Int]`                   |
| Tuples                 | `type MyTuple = (Int, Text)`                |
| Polymorphic types      | `type MyType a b = [(a, b)]`                |

## Data declarations

```daml
-- Record
data MyRecord = MyRecord { label1 : Int, label2 : Text }

-- Product (record syntax with `with`)
data IntAndText = IntAndText with myInt : Int, myText : Text

-- Sum type
data IntOrText = MyInt Int | MyText Text

-- Record with type parameters
data MyRecord a b = MyRecord { label1 : a, label2 : b }

-- Deriving stock instances
data MyRecord = MyRecord { label : Int } deriving (Show, Eq)
```

## Functions

```daml
-- Signature + definition
f : Text -> Text -> Text
f x y = x <> " " <> y

-- Lambda
\x y -> x <> y

-- Polymorphic with constraints
f : (Show a, Eq a) => a -> Text -> Text

-- Application & partial application
f "hello" "world!"
salute : Text -> Text
salute = f "Hello"

-- Functions are first-class
apply : (Text -> Text) -> Text -> Text
apply h x = h x

apply salute "John"   -- "Hello John"
```

## Contract templates

Templates describe data stored on the ledger and the rules for who can read
and write it.

```daml
template MyData
  with
    i      : Int
    party1 : Party
    party2 : Party
    dataKey : (Party, Text)
  where
    signatory party1
    observer  party2
    key dataKey : (Party, Text)
    maintainer key._1

    choice MyChoice : ()
      ...
```

| Keyword       | Meaning                                                                                      |
| ------------- | -------------------------------------------------------------------------------------------- |
| `with`/`where`| Structure the template: `with` lists fields, `where` lists ledger semantics.                 |
| `signatory`   | Party that must authorize creation/archival and grants authority to choices on the contract. |
| `observer`    | Party that can see the contract; no authority.                                               |
| `key`         | A field/expression used as the unique index for contracts of this template.                  |
| `maintainer`  | Parties that guarantee uniqueness of the key; must be derivable from `key`.                  |

## Contract keys

Contract keys are unique, stable references to a contract — stable even when
the underlying `ContractId` changes from an update. They are optional.

- `key` can be any expression over the contract arguments that **does not**
  contain a `ContractId`. It must include all `maintainer` parties.
- `maintainer` parties guarantee per-template key uniqueness. They must be a
  projection of the `key` expression.

## Choices

Choices define how and by whom contract data can be changed.

```daml
(nonconsuming) choice NameOfChoice : ()       -- optional `nonconsuming`, name, return type
  with
    party1 : Party                            -- choice arguments
    party2 : Party
    i      : Int
  controller party1, party2                   -- parties that can exercise this choice
    do                                        -- update executed when exercised
      assert (i == 42)
      create ...
      exercise ...
      return ()
```

| Modifier        | Effect                                                                                          |
| --------------- | ----------------------------------------------------------------------------------------------- |
| `consuming`     | **Default.** The contract is archived by exercising this choice; further exercises fail.        |
| `nonconsuming`  | The contract survives; the choice can be exercised repeatedly.                                  |

## Updates (inside `do` blocks)

Updates compose the transaction that will be committed to the ledger.

```daml
do
  cid <- create NewContract with field1 = 1
                                , field2 = "hello world"
  let answer = 42
  exercise cid SomeChoice with choiceArgument = "123"
  return answer
```

| Action          | Usage                                                                                                       |
| --------------- | ----------------------------------------------------------------------------------------------------------- |
| `create`        | `create NameOfTemplate with exampleParameters`                                                              |
| `exercise`      | `exercise IdOfContract NameOfChoice with choiceArgument1 = value1`                                          |
| `exerciseByKey` | `exerciseByKey @ContractType contractKey NameOfChoice with choiceArgument1 = value1`                        |
| `fetch`         | `c <- fetch IdOfContract`                                                                                   |
| `fetchByKey`    | `c <- fetchByKey @ContractType contractKey`                                                                 |
| `lookupByKey`   | `mCid <- lookupByKey @ContractType contractKey`  *(returns `Optional ContractId`)*                          |
| `abort`         | `abort errorMessage` — abort the transaction with a message                                                 |
| `assert`        | `assert (condition == True)` — fail the transaction if the predicate is false                               |
| `getTime`       | `now <- getTime` — read ledger effective time (never use a wall clock)                                      |
| `return`        | `return 42` — return a value from a `do` block                                                              |
| `let`           | Bind a local variable or function: `let answer = 42`                                                        |
| `this`          | Refer to the current contract inside one of its choices: `create NewContract with owner = this.owner`       |
| `forA`          | Run an action for each element of a list: `forA [alice, bob] $ \p -> create NewContract with owner = p`     |

## Scripts

Daml Script is the scripting language for running Daml commands against a
ledger (in-memory for tests, or a real ledger).

```daml
module Test where

import Daml.Script

test : Script ()
test = do
  alice <- allocateParty "Alice"
  bob   <- allocateParty "Bob"
  c <- submit alice $ createCmd NewContract with ...
  submit bob $ exerciseCmd c Accept with ...
```

Scripts are compiled to the project's `.dar` via `dpm build`.

| Action                                            | Command                                                                                                          |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Run a script                                      | `dpm script --dar example-0.0.1.dar --script-name ModuleName:scriptFn --ledger-host localhost --ledger-port 6865`|
| Run a script with initial JSON arguments          | `dpm script --dar … --input-file args.json --script-name ModuleName:scriptFn --ledger-host … --ledger-port …`    |
| Allocate a party                                  | `alice <- allocateParty "Alice"`                                                                                 |
| List all known parties                            | `parties <- listKnownParties`                                                                                    |
| Query a template visible to a party               | `query @ExampleTemplate alice`                                                                                   |
| Create a contract                                 | `createCmd ExampleTemplate with ...`                                                                             |
| Exercise a choice                                 | `exerciseCmd contractId ChoiceName with ...`                                                                     |
| Exercise a choice by key                          | `exerciseByKeyCmd contractKey ChoiceName with ...`                                                               |
| Create and exercise in one transaction            | `createAndExerciseCmd (ExampleTemplate with ...) (ChoiceName with ...)`                                          |
| Advance ledger time (static-time only)            | `passTime (hours 10)`                                                                                            |
| Set ledger time (static-time only)                | `setTime (time (date 2007 Apr 5) 14 30 05)`                                                                      |

Static time is the default for the in-memory ledger used by Daml Studio and
`dpm test`; wall-clock sandboxes ignore `passTime`/`setTime`.

## JavaScript / React API

Daml ledgers expose a unified JSON API. The TypeScript libraries
`@daml/ledger` and `@daml/react` wrap it for browser apps.

```ts
import Ledger from "@daml/ledger";
import { useParty, useQuery, useLedger, useReload,
         useStreamQuery, useFetchByKey, useStreamFetchByKey,
         DamlLedger } from "@daml/react";
```

React entry point:

```tsx
const App: React.FC = () => (
  <DamlLedger
    token={/* your authentication token */}
    httpBaseUrl={/* optional http base url */}
    wsBaseUrl={/* optional websocket base url */}
    party={/* the logged-in party */}
  >
    <MainScreen />
  </DamlLedger>
);
```

| Action                                                  | Snippet                                                                                                  |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Get the logged-in party                                 | `const party = useParty();`                                                                              |
| Query the ledger                                        | `const { contracts, loading } = useQuery(ContractTemplate, () => ({ field: value }), [dep1, dep2]);`     |
| Query by contract key                                   | `const { contracts, loading } = useFetchByKey(ContractTemplate, () => key, [dep1, dep2]);`               |
| Re-run a query                                          | `const reload = useReload(); /* ... */ onClick={() => reload()}`                                         |
| Streaming query (auto-refreshes)                        | `const { contracts, loading } = useStreamQuery(ContractTemplate, () => ({ field: value }), [dep1]);`     |
| Streaming query by key                                  | `const { contracts, loading } = useStreamFetchByKey(ContractTemplate, () => key, [dep1]);`               |
| Create a contract                                       | `const ledger = useLedger(); const c = await ledger.create(ContractTemplate, arguments);`                |
| Archive a contract                                      | `const ledger = useLedger(); const ev = await ledger.archive(ContractTemplate, contractId);`             |
| Exercise a choice                                       | `const ledger = useLedger(); const [ret, events] = await ledger.exercise(ContractChoice, cid, args);`    |

## What this cheat sheet does *not* cover

The upstream cheat sheet is deliberately syntax-focused. For these topics,
go to the deeper references:

- Pattern matching, `case`/`if`/`let`/`where`, `Optional`/`Either`, type
  classes, stdlib modules → [language.md](language.md).
- `submitMulti`, `submitMustFail`, query helpers, time control patterns,
  multi-package test setups → [daml-script.md](daml-script.md).
- `daml.yaml` / `multi-package.yaml` fields, codegen, sandbox, and the full
  `dpm` command surface → [dpm-cli.md](dpm-cli.md).
- Wallet-integrated dApps via CIP-0103 → [partylayer.md](partylayer.md).
