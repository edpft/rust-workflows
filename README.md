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
| `nix-flake-update.yml` | Runs `nix flake update` and opens a pull request, so inputs do not drift. Dependabot does not understand `flake.lock`. |

```yaml
jobs:
  checks:
    uses: edpft/rust-workflows/.github/workflows/nix-checks.yml@main
```

`nix-checks.yml` and `nix-audit.yml` accept `system` and `runs-on` inputs,
defaulting to `x86_64-linux` on `ubuntu-latest`; override them together.

`nix-flake-update.yml` opens its pull request with the default `GITHUB_TOKEN`,
and GitHub raises no workflow events for that token — so the pull request will
not trigger CI. Rather than requiring a personal access token, it runs
`nix flake check` against the updated inputs itself, before opening anything:
a broken update fails the scheduled run and never becomes a pull request.

Pass a token only if you want the checks to appear on the pull request itself
rather than in the scheduled run, and set `verify: false` to avoid running them
twice:

```yaml
jobs:
  update:
    uses: edpft/rust-workflows/.github/workflows/nix-flake-update.yml@main
    with:
      verify: false
    secrets:
      token: ${{ secrets.FLAKE_UPDATE_TOKEN }}
```

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
