# daml-skill

A [Claude Code](https://claude.com/claude-code) agent skill for developing,
building, and testing **DAML** (Digital Asset Modeling Language) smart contracts
for the [Canton Network](https://docs.canton.network/).

When installed, Claude automatically uses this skill whenever you work with
`.daml` files, `daml.yaml` / `multi-package.yaml`, Daml Script tests, or the
`dpm` (Daml Package Manager) CLI.

## What it covers

- **The DAML language** — templates, choices, contract keys, data types,
  interfaces, pattern matching, and the standard library.
- **Daml Script testing** — multi-party `submit`/`query`/`assert` patterns, time
  control, negative tests, and coverage.
- **The `dpm` CLI** — project scaffolding, `build`, `test`, `sandbox`, codegen,
  `daml.yaml` configuration, upgrades, and troubleshooting.
- **Authorization modeling** — the signatory/observer/controller rules that catch
  out newcomers, and least-authority best practices.

## Installation

The skill source lives in `~/.agents/skills/` and is exposed to Claude Code via a
relative symlink in `~/.claude/skills/`:

```sh
git clone git@github.com:tuddman/daml-skill.git ~/.agents/skills/daml
ln -s ../../.agents/skills/daml ~/.claude/skills/daml
```

Claude Code discovers the skill on its next run. Invoke it explicitly with
`/daml`, or just let it trigger automatically when you touch DAML files.

## Requirements

The skill drives a real toolchain — it does not bundle one:

- **`dpm`** (Daml Package Manager) CLI —
  `curl https://get.digitalasset.com/install/install.sh | sh`
- **JDK 17+** with `JAVA_HOME` set
- **VS Code** with the Daml extension (optional, for `dpm studio`)

`dpm` replaces the older `daml` assistant; commands like `daml build` and
`daml start` no longer exist.

## Repository layout

```
daml/
├── SKILL.md                  # skill entry point — instructions Claude loads
└── references/
    ├── language.md           # DAML language reference
    ├── daml-script.md        # Daml Script testing reference
    └── dpm-cli.md            # dpm CLI + daml.yaml/multi-package.yaml reference
```

`SKILL.md` carries the always-loaded guidance; the `references/` files are
progressively loaded by Claude only when a task needs that depth.

## License

MIT
