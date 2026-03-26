[![Apache 2.0](https://img.shields.io/badge/license-Apache2.0-orange.svg)](./LICENSE)

# Re-usable GitHub actions and workflows
## Actions
| name       | description                                            |
|------------|--------------------------------------------------------|
| taskfile   | invoke task command in github flows                    |
| ghcr-build | prepare buildx / ghcr creds and run `taskfile` command |

## Taskfiles
Re-usable [Taskfile](https://taskfile.dev/) modules via remote includes (Taskfile v3.41+).

| name  | description                     | tasks                                                                                      |
|-------|---------------------------------|--------------------------------------------------------------------------------------------|
| cargo | Rust CI tasks running in Docker | `cargo/fmt`, `cargo/check`, `cargo/clippy`, `cargo/test`, `cargo/audit`, `cargo/audit/fix` |
| go    | Go CI tasks running in Docker   | `go/test`, `go/lint`, `go/fumpt`, `go/vulnerability`, `go/mod/tidy`                        |

### Usage
```yaml
version: '3'
includes:
  cargo:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/main/taskfiles/cargo/Taskfile.yml
    vars:
      rust_version: "1.90.0"
```
```shell
task cargo:cargo/fmt
task cargo:cargo/clippy
task cargo:cargo/test
```

Each task exposes configurable variables with sensible defaults (e.g. `TEST_ARGS`, `CLIPPY_ARGS`).

## Contributing
We're open to any new ideas and contributions.  
Found a bug? Have an idea? We welcome pull requests and issues.
