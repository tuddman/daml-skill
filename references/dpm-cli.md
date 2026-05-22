# dpm CLI & Project Configuration Reference

`dpm` (Daml Package Manager) is the command-line entry point for Canton/DAML
development: SDK management, scaffolding, building, testing, local ledgers, and
codegen. It **replaces the legacy `daml` assistant** — see the migration table
below.

## Installation

| Platform | Command |
|----------|---------|
| macOS / Linux | `curl https://get.digitalasset.com/install/install.sh \| sh` |
| Windows | Run the installer from https://get.digitalasset.com/install/latest-windows.html |
| Manual | Download from https://github.com/digital-asset/dpm/releases |

Prerequisites: **JDK 17+** with `JAVA_HOME` set (Eclipse Adoptium or OpenJDK),
and VS Code for the IDE. The SDK installs under `${HOME}/.dpm/` (override with
`DPM_HOME`). Verify with `dpm --version`.

## SDK version management

| Command | Purpose |
|---------|---------|
| `dpm install` | Download the SDK version pinned in `daml.yaml`. |
| `dpm install 3.4.11` | Install a specific SDK version. |
| `dpm install package` | Install all SDKs required by the current project. |
| `dpm version` | List installed versions (`*` marks the active one). |
| `dpm version --active` | Show the active SDK version. |
| `dpm version --all` | Show all available and installed versions. |
| `dpm version --all -o json` | Machine-readable output. |

The SDK version is per-project (the `sdk-version` field in `daml.yaml`), so the
active version follows the project you are in.

## Project lifecycle

| Command | Purpose |
|---------|---------|
| `dpm init` | Initialize a project in the current directory (creates `daml.yaml`). |
| `dpm new NAME --template T` | Scaffold a new project from a template. |
| `dpm add PACKAGE` | Add a dependency and update `daml.yaml`. |
| `dpm build` | Compile `.daml` sources to a DAR in `.daml/dist/`. |
| `dpm build --all` | Build every package of a multi-package project, in dependency order. |
| `dpm test` | Run all Daml Script tests; prints pass/fail and coverage. |
| `dpm clean` | Remove build artifacts (DARs, `.daml/`). |

Useful templates for `dpm new`: `daml-intro-contracts`, `daml-intro-choices`
(introductory tutorials), plus skeleton/quickstart templates.

## Local ledger & IDE

| Command | Purpose |
|---------|---------|
| `dpm sandbox` | Start a local single-participant Canton node with an in-memory ledger. |
| `dpm canton-console` | Open the Canton admin console (separate terminal). |
| `dpm studio` | Launch VS Code with the Daml extension for the current project. |
| `dpm daml-shell` | Shell client for the Participant Query Store (PQS). |
| `dpm pqs` | Run the Participant Query Store. |

`dpm sandbox` default ports:

| Port | API |
|------|-----|
| 6865 | Ledger API |
| 6866 | Participant Admin API |
| 6867 | Sequencer public API |
| 6868 | Sequencer admin API |
| 6869 | Mediator admin API |

Sandbox options: `--port`, `--admin-api-port`, `--json-api-port`,
`-c|--config FILE`, `--dar PATH` (upload a DAR on startup). For authenticated
runs, pass a JWT config: `dpm sandbox --config auth.conf`.

## Code generation

Generate typed client bindings from a compiled DAR:

| Command | Output |
|---------|--------|
| `dpm codegen-js DAR -o DIR` | TypeScript / JavaScript bindings. |
| `dpm codegen-java DAR -o DIR` | Java bindings. |
| `dpm codegen-alpha-typescript` / `-java` / `-scala` | Newer (alpha) generators. |

```bash
dpm build
dpm codegen-java .daml/dist/my-app-1.0.0.dar -o generated-java
dpm codegen-js   .daml/dist/my-app-1.0.0.dar -o generated-ts
```

## Compiler & DAR utilities

| Command | Purpose |
|---------|---------|
| `dpm damlc` | Invoke the underlying Daml compiler directly. |
| `dpm script` | Run a Daml Script binary against a ledger (`--ledger-host`/`--ledger-port`). |
| `dpm docs` | Generate documentation from `-- |` doc comments. |
| `dpm inspect-dar DAR` | Inspect the contents of a DAR archive. |
| `dpm validate-dar DAR` | Validate a DAR archive. |
| `dpm upgrade-check` | Check package upgrade/version compatibility before release. |

## `daml.yaml`

Every project has a `daml.yaml` at its root:

```yaml
sdk-version: 3.4.0          # SDK version (dpm install fetches it)
name: my-app                # package name (used in the DAR filename)
version: 1.0.0              # package version (bump for releases / upgrades)
source: daml                # directory containing .daml sources
dependencies:
  - daml-prim              # always present
  - daml-stdlib            # always present
  - daml-script            # required only if the package has Daml Script tests
build-options:
  - --target=3.4           # Daml-LF target version
```

| Field | Meaning |
|-------|---------|
| `sdk-version` | SDK version; must be installed (`dpm install`). |
| `name` | Package name; appears in the DAR filename. |
| `version` | Package version; bump it for every release so upgrade checks work. |
| `source` | Source root directory (modules are paths relative to it). |
| `dependencies` | Library and DAR dependencies. |
| `build-options` | Compiler flags, e.g. the Daml-LF `--target`. |

Keep `--target` / `sdk-version` consistent with any DAR dependencies — a Daml-LF
target mismatch is a common, confusing build failure.

## `multi-package.yaml`

For repositories with several DAML packages, place a `multi-package.yaml` at the
root:

```yaml
packages:
  - ./contracts
  - ./tests
  - ./scripts
```

Then `dpm build --all` compiles every package in dependency order.

## `dpm-config.yaml`

Global dpm configuration lives at `${DPM_HOME}/dpm-config.yaml` — registry URL
overrides, authentication credentials, insecure-registry settings. Both
`daml.yaml` and `dpm-config.yaml` support environment-variable interpolation, so
secrets need not be hardcoded.

## Typical workflow

```bash
dpm new my-app --template daml-intro-contracts
cd my-app
dpm install                  # fetch the SDK pinned in daml.yaml

# edit-compile-test loop
dpm build                    # type-check + produce .daml/dist/my-app-1.0.0.dar
dpm test                     # run Daml Script tests + coverage

# local integration testing
dpm sandbox                  # ledger on :6865 (separate terminal)

# client bindings
dpm codegen-java .daml/dist/my-app-1.0.0.dar -o generated-java
```

## Migration from the legacy `daml` assistant

| Legacy `daml` command | Replacement |
|-----------------------|-------------|
| `daml new` | `dpm new` |
| `daml build` | `dpm build` |
| `daml test` | `dpm test` |
| `daml install` | `dpm install` |
| `daml studio` | `dpm studio` |
| `daml sandbox` | `dpm sandbox` |
| `daml codegen java` | `dpm codegen-java` |
| `daml codegen js` | `dpm codegen-js` |
| `daml start` | `dpm sandbox` + `dpm build` + DAR upload / party allocation via API |

Removed entirely — use the Declarative API, Canton Console, JSON API, or gRPC
Ledger API instead: `daml ledger allocate-parties`, `daml ledger list-parties`,
`daml ledger upload-dar`, `daml ledger fetch-dar`, `daml packages`.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Cannot run command without SDK` | `dpm install`, or fix `sdk-version` in `daml.yaml`. |
| Port already in use (sandbox) | Pass `--port` / `--admin-api-port` / etc. |
| Build fails on a dependency | Align Daml-LF `--target` / `sdk-version` across packages. |
| `JAVA_HOME` / JDK errors | Install JDK 17+ and export `JAVA_HOME`. |
| Authorization errors on sandbox | Start with `dpm sandbox --config auth.conf` (JWT settings). |
| Stale build artifacts | `dpm clean`, then `dpm build`. |

## CI/CD

A minimal pipeline: install the SDK, build, test.

```bash
dpm install
dpm build --all
dpm test
```

`dpm version --all -o json` and `dpm test` produce machine-readable output
suitable for CI gating. Cache `${DPM_HOME}` between runs to avoid re-downloading
the SDK.
