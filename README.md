[![Apache 2.0](https://img.shields.io/badge/license-Apache2.0-orange.svg)](./LICENSE)

# Re-usable GitHub Actions, Workflows, and Taskfiles

Shared CI/CD infrastructure for solti projects.

- **Workflows** — reusable workflows in `.github/workflows/`, called via `uses:` from repo workflows.
- **Actions** — composite GitHub actions, consumed from workflow steps via `uses:`.
- **Taskfiles** — [Taskfile](https://taskfile.dev/) modules pulled in as remote includes (requires Taskfile **v3.41+**).

Workflows and actions are referenced by the `v1` tag. Move the tag to release changes.

---

## Reusable workflows

| Workflow           | Purpose                                                                              |
|--------------------|---------------------------------------------------------------------------------------|
| `rust-ci.yml`      | Full Rust CI: fmt, check, unit + integration tests, clippy feature matrix, audit, docs, example builds, publish dry-run, `gate` aggregator |
| `rust-release.yml` | Rust CD for single-crate repos: verify tag-on-main + version==tag, publish, GitHub release |
| `label-check.yml`  | Verify the PR carries one changelog label from the caller's `.github/release.yml`    |

`rust-ci.yml` inputs: `preflight-required` (default `true`; set `false` in workspaces — the publish dry-run legitimately fails on cross-crate version bumps).
`rust-release.yml` inputs: `crate`; secrets: `crates-io-token`.

Usage (repo `pr.yml`):

```yaml
jobs:
  ci:
    uses: soltiHQ/actions/.github/workflows/rust-ci.yml@v1
    with: { preflight-required: true }
```

The branch-protection check to require is `ci / gate` (plus `label-check / required` from `label-check.yml`).

---

## Common

### Actions

| Name         | Purpose                                                                        |
|--------------|--------------------------------------------------------------------------------|
| `taskfile`   | Install Taskfile, export env vars, run a `task <cmd>`                          |
| `ghcr-build` | Set up buildx, log in to GHCR, then run a `task <cmd>`                         |
| `gate`       | Aggregator: fail unless every job in `toJSON(needs)` succeeded (`allow`, `allow-skipped` escape hatches) |

---

## Rust

### Actions

| Name            | Purpose                                                                                      |
|-----------------|----------------------------------------------------------------------------------------------|
| `cargo-publish` | Publish crates to crates.io in order. Soft-exit on `already exists`, retry on HTTP 429.      |
| `cargo-cache`   | Cache `.cache/cargo` and `.cache/target`, keyed by `Cargo.toml` + `rust-toolchain.toml`      |

### Taskfile module

`taskfiles/rust/Taskfile.yml`: Rust CI tasks running inside a sandboxed Docker image
(`ghcr.io/soltihq/rust-ci:<rust-version>`, toolchain pinned to the caller's `rust-version`).

| Task              | Command                                                  |
|-------------------|-----------------------------------------------------------|
| `fmt`             | `cargo fmt --check --verbose`                             |
| `check`           | `cargo check`                                             |
| `build`           | `cargo build -p <CRATE>` (requires `CRATE`)               |
| `clippy`          | `cargo clippy --all --all-features -- -D warnings`        |
| `test`            | `cargo test --all --all-features`                         |
| `bench`           | `cargo bench`                                             |
| `audit`           | `cargo audit`                                             |
| `publish-dry-run` | `cargo publish --dry-run -p <CRATE>` (requires `CRATE`)   |
| `docs`            | `rustdoc` in docs.rs emulation mode (nightly)             |
| `fmt/fix`         | `cargo fmt` — manual                                      |
| `audit/fix`       | `cargo audit fix` — manual                                |

Defaults for arg-like variables (`FMT_ARGS`, `CHECK_ARGS`, `CLIPPY_ARGS`, `TEST_ARGS`) can be overridden per call.

#### Usage

```yaml
# Taskfile.yml
version: '3'
includes:
  rust:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/main/taskfiles/rust/Taskfile.yml
    vars:
      image_patch: "-1"
```

```shell
task rust:fmt
task rust:clippy
task rust:test
task rust:publish-dry-run CRATE=my-crate
```

---

## Go

### Taskfile module

`taskfiles/go/Taskfile.yml`: Go CI tasks running inside a sandboxed Docker image.

| Task               | Command                                   |
|--------------------|-------------------------------------------|
| `go/test`          | `go test ./...`                           |
| `go/lint`          | `golangci-lint run ./...`                 |
| `go/fumpt`         | `gofumpt -l -w` over all tracked `.go`    |
| `go/vulnerability` | `govulncheck ./...`                       |
| `go/mod/tidy`      | `go mod tidy`                             |

#### Usage

```yaml
# Taskfile.yml
version: '3'
includes:
  ci:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/main/taskfiles/go/Taskfile.yml
    vars:
      go_version: "1.23"
```

```shell
task ci:go/test
task ci:go/lint
```

---

## Proto

### Taskfile module

`taskfiles/proto/Taskfile.yml`: Protobuf CI tasks running `buf` and `clang-format` inside a sandboxed Docker image (`ghcr.io/soltihq/buf-ci`). 
Code generation is intentionally left to consumers; this module only validates the schema.

| Task          | Command                                                   |
|---------------|-----------------------------------------------------------|
| `lint`        | `buf lint`                                                |
| `build`       | `buf build` (compile the schema)                          |
| `format`      | `clang-format --dry-run --Werror` over `*.proto` (check)  |
| `breaking`    | `buf breaking --against <main>` (override with `AGAINST`) |
| `format/fix`  | `clang-format -i` over `*.proto` — manual                 |

The image version is pinned with `buf_version` (there is no manifest like `Cargo.toml`/`go.mod` to read it from); 
`breaking` compares against `main` by default.

#### Usage

```yaml
# Taskfile.yml
version: '3'
includes:
  proto:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/main/taskfiles/proto/Taskfile.yml
    vars:
      buf_version: "1.50.0"
      image_patch: "-1"
```

```shell
task proto:lint
task proto:breaking
task proto:breaking AGAINST=https://github.com/soltiHQ/proto.git#branch=main
```

---

## Contributing

Found a bug? Have an idea? Pull requests and issues welcome.
