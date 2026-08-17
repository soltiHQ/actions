[![Apache 2.0](https://img.shields.io/badge/license-Apache2.0-orange.svg)](./LICENSE)

# Solti Actions

Reusable CI/CD workflows, composite actions, and Taskfile modules for Solti repositories.

Consumer repositories keep their project-specific commands and release configuration.
This repository owns the shared GitHub Actions machinery and toolchain containers.

| [Quick start](#quick-start) | [Workflows](#reusable-workflows) | [Actions](#composite-actions) | [Taskfiles](#taskfile-modules) |

## Contract

| Component              | Location             | Used from                                      |
|------------------------|----------------------|------------------------------------------------|
| Reusable workflows     | `.github/workflows/` | A caller workflow job through `uses`           |
| Composite actions      | `<name>/action.yml`  | A workflow step through `uses`                 |
| Taskfile modules       | `taskfiles/`         | A repository `Taskfile.yml` through `includes` |

Workflows and actions use the `v1` tag.
Remote Taskfile includes should use the same tag.

The Taskfile modules run tools inside versioned images from [`soltiHQ/images`](https://github.com/soltiHQ/images):

| Module    | Image                                     | Version source                        |
|-----------|-------------------------------------------|---------------------------------------|
| Rust      | `ghcr.io/soltihq/ci/rust:<version>`       | Root `Cargo.toml` `rust-version`      |
| Go        | `ghcr.io/soltihq/ci/golang:<version>`     | Root `go.mod` `go` directive          |
| Proto     | `ghcr.io/soltihq/ci/proto:<version>`      | `buf_version`, defaulting to `1.50.0` |
| Node      | `ghcr.io/soltihq/ci/node:<version>`       | `app/.node-version`                   |
| Terraform | `ghcr.io/soltihq/ci/terraform:<version>`  | `tf/.terraform-version`               |
| AWS       | `ghcr.io/soltihq/ci/aws:<version>`        | Root `.aws-cli-version`               |

CI refreshes the selected image before each task.
Local runs reuse an existing image and pull it when missing.

## Quick start

A Rust repository exposes its own `ci/*` tasks through `Taskfile.yml`:

```yaml
version: '3'

includes:
  rust:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/v1/taskfiles/rust/Taskfile.yml

tasks:
  ci/fmt:
    cmds:
      - task: rust:fmt

  ci/check:
    cmds:
      - task: rust:check
        vars: { CHECK_ARGS: '--all-targets --all-features --locked' }
```

The pull-request workflow delegates CI to the shared workflow:

```yaml
name: PR

on:
  pull_request:

jobs:
  ci:
    uses: soltiHQ/actions/.github/workflows/rust-ci.yml@v1
```

For a Cargo workspace, enable workspace-specific checks:

```yaml
jobs:
  ci:
    uses: soltiHQ/actions/.github/workflows/rust-ci.yml@v1
    with:
      workspace: true
```

The branch-protection check is `ci / gate` when the caller job is named `ci`.

## Reusable workflows

| Workflow                                                                     | Purpose                                                             |
|------------------------------------------------------------------------------|---------------------------------------------------------------------|
| [`rust-ci.yml`](.github/workflows/rust-ci.yml)                               | Validate a Rust package or workspace                                |
| [`rust-release.yml`](.github/workflows/rust-release.yml)                     | Publish one crate and create one GitHub release                     |
| [`rust-workspace-release.yml`](.github/workflows/rust-workspace-release.yml) | Publish workspace crates in dependency order and create one release |
| [`label-check.yml`](.github/workflows/label-check.yml)                       | Require a changelog label declared in `.github/release.yml`         |
| [`static-ci.yml`](.github/workflows/static-ci.yml)                           | Build a static site and validate its Terraform stack                |
| [`static-release.yml`](.github/workflows/static-release.yml)                 | Apply infrastructure, upload a tagged site, and invalidate its CDN  |

> TODO: fix it;  Both Rust release workflows install `protoc` before Cargo verifies and publishes package tarballs.

### Rust CI

`rust-ci.yml` calls the consumer repository's `ci/*` tasks.
The shared workflow owns job isolation, caching, matrices, and the final gate.

| Job               | Consumer task or behavior                                      |
|-------------------|----------------------------------------------------------------|
| `fmt`             | `ci/fmt`                                                       |
| `MSRV`            | `ci/check`                                                     |
| `unittest`        | `ci/test-unit`                                                 |
| `integration`     | `ci/test-integration`                                          |
| `clippy`          | `ci/clippy FEATURE=<configuration>`                            |
| `audit`           | `ci/audit`                                                     |
| `docs`            | `ci/docs`                                                      |
| `examples-build`  | `ci/build CRATE=<package>` for packages containing examples    |
| `package`         | `ci/package` when `workspace: true`                            |
| `preflight`       | Advisory `ci/publish-dry-run`; not included in the final gate  |
| `gate`            | Require every non-advisory CI dependency to succeed            |

Package repositories test `none`, every declared feature, and `all` in separate Clippy jobs.
Workspace repositories test `none` and `all` across the workspace.

A workspace can add focused package configurations:

```toml
[workspace.metadata.ci]
clippy-matrix = [
    { package = "my-crate", features = ["feature-a", "feature-b"] },
]
```

Each entry must name an existing workspace package and existing features.

### Labels

`label-check.yml` reads allowed changelog and exclusion labels from the caller's `.github/release.yml`.
The pull request must carry at least one of them.

```yaml
jobs:
  label-check:
    uses: soltiHQ/actions/.github/workflows/label-check.yml@v1
```

The branch-protection check is `label-check / required` when the caller job is named `label-check`.

### Static site CI and release

`static-ci.yml` runs the consumer's code build, dependency audit, and Terraform format/validate checks.
`static-release.yml` verifies that the tag belongs to the default branch, builds one artifact, applies Terraform,
exports the resulting infrastructure outputs, uploads the artifact to the resolved S3 bucket, invalidates CloudFront,
and creates the GitHub release.

The release workflow receives infrastructure coordinates as typed inputs.
AWS authentication uses a role assumed through GitHub OIDC; consumers do not pass long-lived AWS access keys.
Terraform owns infrastructure state and output generation. The deploy job receives only the resolved bucket,
CloudFront distribution ID, and AWS region before invoking the consumer's AWS deployment task.

### Single-crate release

`rust-release.yml` verifies that the tagged commit belongs to `main` and that `package.version` matches the tag.
It then publishes the crate and creates a GitHub release.

```yaml
jobs:
  publish:
    uses: soltiHQ/actions/.github/workflows/rust-release.yml@v1
    with:
      crate: my-crate
    secrets:
      crates-io-token: ${{ secrets.CRATES_IO_TOKEN }}
```

### Workspace release

`rust-workspace-release.yml` reads publishable crates from `.github/crates.txt` by default.
The tagged commit must belong to `default-branch`, which defaults to `main`.
The file must contain every publishable workspace crate exactly once and in dependency order.
Every published crate version must match the tag.

```yaml
jobs:
  publish:
    uses: soltiHQ/actions/.github/workflows/rust-workspace-release.yml@v1
    with:
      crates-file: .github/crates.txt
      prepare-task: proto/vendor
      allow-dirty: true
    secrets:
      crates-io-token: ${{ secrets.CRATES_IO_TOKEN }}
```

`prepare-task` is optional.
Use `allow-dirty` only when that task creates required package inputs.

## Composite actions

| Action                                      | Purpose                                                                    |
|---------------------------------------------|----------------------------------------------------------------------------|
| [`taskfile`](taskfile/action.yml)           | Install Task, export optional variables, and run one repository task       |
| [`gate`](gate/action.yml)                   | Validate the results supplied through `toJSON(needs)`                      |
| [`cargo-cache`](cargo-cache/action.yml)     | Cache `.cache/cargo` and `.cache/target` under a caller-provided scope     |
| [`cargo-publish`](cargo-publish/action.yml) | Publish crates in order, tolerate existing versions, and retry HTTP 429    |
| [`ghcr-build`](ghcr-build/action.yml)       | Build and publish a multi-platform GHCR image with version and commit tags |

The `taskfile` action installs Task `3.44.1` by default.
Set its `version` input to override the binary version.

`gate` accepts two explicit exceptions:

- `allow` tolerates any result for named jobs;
- `allow-skipped` tolerates only `skipped` for named jobs.

`ghcr-build` targets `linux/amd64` and `linux/arm64` by default.
It publishes `<tag>` and `<tag>-sha-<commit>`.

## Taskfile modules

The modules provide low-level tasks.
Consumer repositories wrap them with stable, repository-specific commands such as `ci/test` or `proto/vendor`.
Every tool module includes the shared Docker module internally and delegates container execution to it.

### [Rust](taskfiles/rust/Taskfile.yml)

| Module task       | Operation                                                   |
|-------------------|-------------------------------------------------------------|
| `fmt`             | Check `cargo fmt`                                           |
| `check`           | Run `cargo check`                                           |
| `build`           | Build one package selected by `CRATE`                       |
| `clippy`          | Run Clippy with warnings denied                             |
| `test`            | Run Cargo tests                                             |
| `bench`           | Run Cargo benchmarks                                        |
| `audit`           | Run `cargo audit`                                           |
| `package-list`    | Inspect the package file set for `CRATE`                    |
| `publish-dry-run` | Simulate publishing `CRATE`                                 |
| `doc`             | Build stable rustdoc with warnings denied                   |
| `docs`            | Emulate the docs.rs nightly rustdoc build                   |
| `fmt/fix`         | Apply `cargo fmt`                                           |
| `audit/fix`       | Apply supported `cargo audit fix` changes                   |

Argument variables such as `CHECK_ARGS`, `CLIPPY_ARGS`, and `TEST_ARGS` let the consumer define its exact repository policy.

### [Go](taskfiles/go/Taskfile.yml)

```yaml
includes:
  go:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/v1/taskfiles/go/Taskfile.yml

tasks:
  ci/test:
    cmds:
      - task: go:test

  ci/lint:
    cmds:
      - task: go:golangci
```

| Module task   | Operation                                                 |
|---------------|-----------------------------------------------------------|
| `gofumpt`     | Check tracked Go files                                    |
| `golangci`    | Run `golangci-lint`                                       |
| `build`       | Build a named binary for the selected `GOOS` and `GOARCH` |
| `govulncheck` | Run Go vulnerability analysis                             |
| `test`        | Run Go tests                                              |
| `proto`       | Generate protobuf sources with Buf                        |
| `templ`       | Generate templ sources                                    |
| `tailwindcss` | Build CSS from required `INPUT` and `OUTPUT` paths        |
| `tidy`        | Run `go mod tidy`                                         |
| `vendor`      | Run `go mod vendor`                                       |
| `fmt`         | Apply gofumpt                                             |

### [Proto](taskfiles/proto/Taskfile.yml)

```yaml
includes:
  proto:
    taskfile: https://raw.githubusercontent.com/soltiHQ/actions/v1/taskfiles/proto/Taskfile.yml

tasks:
  ci/lint:
    cmds:
      - task: proto:lint
```

| Module task  | Operation                                                       |
|--------------|-----------------------------------------------------------------|
| `lint`       | Run `buf lint`                                                  |
| `build`      | Compile the schema with `buf build`                             |
| `format`     | Check every `.proto` file with `clang-format`                   |
| `breaking`   | Compare against `.git#branch=main` or an explicit `AGAINST`     |
| `format/fix` | Apply `clang-format`                                            |

Code generation remains a consumer responsibility.
The Proto module validates the schema.

### [Node](taskfiles/node/Taskfile.yml)

The Node module derives its image tag from `app/.node-version` and exposes internal `type-check`, `build`, and `audit` tasks.
Each command installs the exact lockfile with `npm ci` inside the pinned `ghcr.io/soltihq/ci/node` image.

### [Terraform](taskfiles/terraform/Taskfile.yml)

The Terraform module derives its image tag from `tf/.terraform-version`. It exposes internal format, validate, and apply
tasks. Remote operations initialize the S3 backend with its lockfile. The static release workflow reads the resulting
Terraform outputs directly after apply and passes the selected values to the deployment job.

### [AWS](taskfiles/aws/Taskfile.yml)

The AWS module derives its image tag from the repository-root `.aws-cli-version`. It exposes internal `s3/publish`
and `cloudfront/invalidate` tasks. `s3/publish` uploads immutable assets, uploads `index.html` with no-cache metadata,
and removes stale non-index objects. `cloudfront/invalidate` creates a full distribution invalidation and waits for it.

### [Helpers](taskfiles/helpers/Taskfile.yml)

`taskfiles/helpers/Taskfile.yml` provides two internal building blocks:

- `download/file` downloads one missing file;
- `proto/vendor` checks out a selected `soltiHQ/proto` revision and replaces mapped destination trees.

Consumers decide the protobuf revision and source-to-destination mappings.

## Versioning

Use `@v1` for workflows and composite actions.
Use `/v1/` in raw Taskfile URLs.

Moving the `v1` tag updates every consumer on its next run.
Test shared changes from an explicit branch or commit before moving the tag.

Toolchain image versions are independent of the actions revision.
They come from `Cargo.toml`, `go.mod`, `app/.node-version`, `tf/.terraform-version`, the repository-root
`.aws-cli-version`, or the explicit Proto setting.

## Contributing

Issues and pull requests are welcome.

Changes to reusable workflows, actions, and Taskfile modules affect their consumers independently.
Test the boundary being changed from a representative caller repository.

Read the [contributing guide](https://github.com/soltiHQ/.github/blob/main/CONTRIBUTING.md) before a large change.

<br>

<p align="center">
  <a href="https://github.com/soltiHQ">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/soltiHQ/.github/main/assets/word/solti-word-light.svg">
      <img src="https://raw.githubusercontent.com/soltiHQ/.github/main/assets/logo/solti-logo-dark.svg" alt="soltiHQ" height="84">
    </picture>
  </a>
</p>
