[![Apache 2.0](https://img.shields.io/badge/license-Apache2.0-orange.svg)](./LICENSE)

# Re-usable GitHub Actions and Taskfiles

Shared CI/CD infrastructure for solti projects.

- **Actions** — composite GitHub actions, consumed from workflows via `uses:`.
- **Taskfiles** — [Taskfile](https://taskfile.dev/) modules pulled in as remote includes (requires Taskfile **v3.41+**).

All modules are referenced by `@main`.

---

## Common

### Actions

| Name         | Purpose                                                              |
|--------------|----------------------------------------------------------------------|
| `taskfile`   | Install Taskfile, export env vars, run a `task <cmd>`                |
| `ghcr-build` | Set up buildx, log in to GHCR, then run a `task <cmd>`               |
| `cache-get`  | Restore a cache by `paths` + `key` (wraps `actions/cache/restore`)   |
| `cache-put`  | Save a cache by `paths` + `key` (wraps `actions/cache/save`)         |

---

## Rust

### Actions

| Name            | Purpose                                                                                      |
|-----------------|----------------------------------------------------------------------------------------------|
| `cargo-publish` | Publish a crate to crates.io. Soft-exit on `already exists`, retry on HTTP 429 rate-limit.   |

### Taskfile module

`taskfiles/cargo/Taskfile.yml`: Rust CI tasks running inside a sandboxed Docker image.

| Task                    | Command                                                    |
|-------------------------|------------------------------------------------------------|
| `cargo/fmt`             | `cargo fmt --check --verbose`                              |
| `cargo/check`           | `cargo check`                                              |
| `cargo/clippy`          | `cargo clippy --all --all-features -- -D warnings`         |
| `cargo/test`            | `cargo test --all --all-features`                          |
| `cargo/audit`           | `cargo audit`                                              |
| `cargo/publish-dry-run` | `cargo publish --dry-run -p <CRATE>` (requires `CRATE`)    |
| `cargo/audit/fix`       | `cargo audit fix` — manual                                 |

Defaults for arg-like variables (`FMT_ARGS`, `CHECK_ARGS`, `CLIPPY_ARGS`, `TEST_ARGS`, `PUBLISH_ARGS`) can be overridden per call.

#### Usage

```yaml
# Taskfile.yml
version: '3'
includes:
  ci:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/main/taskfiles/cargo/Taskfile.yml
    vars:
      rust_version: "1.90.0"
```

```shell
task ci:cargo/fmt
task ci:cargo/clippy
task ci:cargo/test
task ci:cargo/publish-dry-run CRATE=my-crate
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

## Contributing

Found a bug? Have an idea? Pull requests and issues welcome.
