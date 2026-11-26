# security-pipeline

🇷🇺 [Русская версия](README_RUS.md)

A reusable workflow that scans a repository for secrets and IaC misconfigurations with gitleaks, kics and checkov. The scanners run from signed images in the Zot registry of the [as-platform](https://github.com/as-platform) homelab platform, on the ephemeral runner `["self-hosted","homelab"]`, without docker: the images are verified with `cosign`, pulled with `buildah from` and started with `buildah run`.

The scanners always run in report mode. One bash and `jq` step reads their results and decides whether the job fails, so a scan never breaks a pipeline until the caller asks for it.

## What it runs

| Tool       | Practice       | Mode                                                          | `gated` is                              |
| ---------- | -------------- | ------------------------------------------------------------- | --------------------------------------- |
| `gitleaks` | `SAST/SECRETS` | `detect --no-git --redact`, working tree only                 | every finding                           |
| `kics`     | `SAST/IAC`     | `scan --ignore-on-exit results --disable-secrets`            | findings at or above `severity-threshold` |
| `checkov`  | `SAST/IAC`     | `--soft-fail --skip-framework secrets --skip-download`        | every failed check                      |

Secrets are the job of gitleaks only: kics and checkov have their secret scanning turned off, so a found secret is never printed by three tools.

### What it finds

- `gitleaks`: secrets in the working tree, with the default ruleset plus the `.gitleaks.toml` and `.gitleaksignore` of the scanned repository. The git history is not scanned.
- `kics` and `checkov`: misconfigurations of infrastructure as code, such as Kubernetes manifests, Terraform, Dockerfiles and GitHub Actions workflows, and the other formats these tools support.

## Quick start

```yaml
name: security
on:
  push: { branches: [main] }
  pull_request:

permissions:
  contents: read

jobs:
  security:
    if: github.event.pull_request.head.repo.fork != true
    uses: as-platform/security-pipeline/.github/workflows/security-pipeline.yml@main
    secrets:
      zot-user: ${{ secrets.ZOT_USER }}
      zot-password: ${{ secrets.ZOT_PASSWORD }}
```

The calling repository needs access to the runner (the runner group of the organization) and the secrets `ZOT_USER` and `ZOT_PASSWORD`. Pass the secrets explicitly, never with `secrets: inherit`. The workflow is referenced by `@main`, like every other template of the platform.

To run it by hand only, keep `workflow_dispatch` as the single trigger:

```yaml
name: security
on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  security:
    uses: as-platform/security-pipeline/.github/workflows/security-pipeline.yml@main
    secrets:
      zot-user: ${{ secrets.ZOT_USER }}
      zot-password: ${{ secrets.ZOT_PASSWORD }}
```

## Inputs

| Input                    | Default                    | Meaning                                                                                 |
| ------------------------ | -------------------------- | --------------------------------------------------------------------------------------- |
| `runner`                 | `["self-hosted","homelab"]` | JSON array of `runs-on` labels                                                          |
| `timeout-minutes`        | `30`                       | job timeout, keep it below the `activeDeadlineSeconds` of the runner                     |
| `repository`             | `""`                       | `owner/name` of a public repository to scan, empty means the caller repository          |
| `ref`                    | `""`                       | branch, tag or SHA, empty means the ref of the event (the default branch for another repository) |
| `artifact-name`          | `security-pipeline-reports` | name of the artifact with the reports                                                   |
| `severity-threshold`     | `HIGH`                     | one of `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`; applies to kics only                |
| `image-gitleaks`         | pinned, see [Image pins](docs/images.md) | image pin                                                                               |
| `image-kics`             | pinned, see [Image pins](docs/images.md) | image pin                                                                               |
| `image-checkov`          | pinned, see [Image pins](docs/images.md) | image pin                                                                               |
| `disable-gitleaks`       | `false`                    | skip the tool                                                                           |
| `disable-kics`           | `false`                    | skip the tool                                                                           |
| `disable-checkov`        | `false`                    | skip the tool                                                                           |
| `raise-failure`          | `false`                    | fail the job when any enabled tool errors or has gated findings                         |
| `gitleaks-raise-failure` | `false`                    | the same for gitleaks only                                                              |
| `kics-raise-failure`     | `false`                    | the same for kics only                                                                  |
| `checkov-raise-failure`  | `false`                    | the same for checkov only                                                               |

There are no group switches such as `disable-sast` or `disable-iac`, only per tool ones. The registry `zot.homelab` is fixed, there is no `registry` input.

## Secrets

| Secret         | Meaning                          |
| -------------- | -------------------------------- |
| `zot-user`     | Zot user that can read `golden/**` |
| `zot-password` | password of that user            |

Both are required. An empty value fails the `prepare` step with a message about fork pull requests: GitHub does not pass secrets to them, and the pipeline fails closed instead of skipping the scan silently.

## Outputs

| Output    | Meaning                                                                                                              |
| --------- | -------------------------------------------------------------------------------------------------------------------- |
| `results` | compact JSON object per tool: `{"gitleaks":{"status":"ok","findings":4,"gated":4,"incomplete":0,"raise":false},"kics":{...},...}` |
| `verdict` | `fail` when a tool tripped the gate or the job failed before the gate, otherwise `pass`                               |

`status` is `ok`, `error` or `skipped`. Read the outputs in a later job through `needs.<job>.outputs.results` with `fromJSON`, see `examples/lvl_5_blocking.yml`.

What `incomplete` and `gated` mean, and when the gate trips: [Gate semantics](docs/gate.md).

## Examples

| File                           | What it shows                                                                          |
| ------------------------------ | -------------------------------------------------------------------------------------- |
| `examples/lvl_2_base.yml`      | soft mode on push, pull request and manual run, with the fork guard                    |
| `examples/lvl_5_blocking.yml`  | `raise-failure: true` and a deploy job that waits for the gate and reads `results`     |
| `examples/scan_external_repo.yml` | manual scan of another public repository through `repository` and `ref`             |

The examples live outside `.github/workflows/` so they do not run here. `examples/scan_external_repo.yml` scans a repository you do not control: read [Scanning a repository you do not control](docs/security.md#scanning-a-repository-you-do-not-control) first.

## Boundary with ci-templates

[`ci-templates/security.yml`](https://github.com/as-platform/ci-templates) scans the git history with gitleaks and runs `trivy fs`, hadolint and semgrep with the binaries of the runner host. This workflow runs signed images, scans the working tree only and gates on its own results. Call one of the two in a repository, not both. `trivy config` is not part of this workflow.

## Limitations

- gitleaks runs with `--no-git` over a checkout of depth 1: a secret that is removed from the tree but still in the history is not found. Rotate such a secret, and scan the history with `ci-templates/security.yml` where it matters.
- There is no network isolation. The scanners work offline only because of their flags (`--skip-download` for checkov, queries inside the kics image); the runner does not block their traffic.
- Only public repositories can be scanned through `repository`; there is no checkout token.
- `concurrency` is not set, the caller decides. The runner pool is small (`maxReplicaCount 3`), so a pipeline on every push takes one pod per run.
- A sub directory of a monorepo cannot be scanned on its own, the whole checkout is scanned.
- The Zot user in use today can also write to Zot. Connect only repositories you trust with it until a read-only user for `golden/**` exists.

## Documentation

| Document                                  | What it covers                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------------- |
| [`docs/gate.md`](docs/gate.md)             | `raise-failure`, `status`, `gated`, `incomplete`, the result files             |
| [`docs/reports.md`](docs/reports.md)       | the files of the artifact and the allowlists that keep secret values out        |
| [`docs/architecture.md`](docs/architecture.md) | layout, step order, scanner sandbox, why one job                            |
| [`docs/images.md`](docs/images.md)         | image pins and how to bump them                                                |
| [`docs/security.md`](docs/security.md)     | suppressions, security notes, scanning a repository you do not control         |

## Related repositories

- [`ci-templates`](https://github.com/as-platform/ci-templates) — `actions/zot-login` and the workflows this pipeline sits next to
- [`golden-images`](https://github.com/as-platform/golden-images) — builds and signs the scanner images
- [`runner-smoke`](https://github.com/as-platform/runner-smoke) — runs the end to end checks of the pipeline on the runner
- [`k3s`](https://github.com/as-platform/k3s) — the GitOps repository with the runner and Zot

## License

MIT — see [`LICENSE`](LICENSE).
