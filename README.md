# rust-workflows

A repo for storing re-usable workflows.

There are two families, and a repo should pick one.

## Nix repos

For repos whose flake defines its own `checks` — the flake is the source of
truth, and these workflows just run whatever it declares.

| Workflow | What it does |
| --- | --- |
| `nix-checks.yml` | Evaluates `.#checks.<system>` and builds each one as a separate matrix job. Adding a check to `flake.nix` adds a job here. |
| `nix-audit.yml` | Refreshes the `advisory-db` flake input, then builds the caller's audit check. For scheduled runs; the pinned check already runs under `nix-checks.yml`. |

```yaml
jobs:
  checks:
    uses: edpft/rust-workflows/.github/workflows/nix-checks.yml@main
```

Both accept `system` and `runs-on` inputs, defaulting to `x86_64-linux` on
`ubuntu-latest`; override them together.

## Cargo repos

For repos without a flake. Each installs a toolchain with
`dtolnay/rust-toolchain` and runs one cargo command.

| Workflow | What it does |
| --- | --- |
| `check-formatting.yml` | `cargo fmt --all --check` |
| `lint.yml` | `cargo clippy --locked --all-targets -- -D warnings` |
| `test.yml` | `cargo test --locked` |
| `audit.yml` | `cargo audit`, with a prebuilt `cargo-audit` |

`dtolnay/rust-toolchain` does **not** read `rust-toolchain.toml`. These default
to `stable`, so a repo with a pinned toolchain must pass it explicitly or CI
will drift from local builds:

```yaml
jobs:
  test:
    uses: edpft/rust-workflows/.github/workflows/test.yml@main
    with:
      toolchain: "1.95.0"
```

Do not mix these with `nix-checks.yml` — they would run the same checks twice,
on a different toolchain.

## Any repo

| Workflow | What it does |
| --- | --- |
| `auto-merge.yml` | Enables auto-merge on Dependabot pull requests. Major bumps are excluded unless `allow-major: true`; `merge-method` defaults to `merge`. |
| `release-please.yml` | Runs release-please. All inputs optional; `release-type` defaults to `rust`. Outputs `release-created` and `tag-name` for gating a publish job. Pass a `token` secret if a release must trigger further workflows. |

`release-please.yml` still accepts `package-name`, but the action dropped that
input at v4 and it is ignored — set `$.packages[path].package-name` in a
`release-please-config.json` instead. The input is kept only so that existing
callers do not fail; new callers should omit it.
