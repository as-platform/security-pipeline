# security-pipeline

🇷🇺 [Русская версия](README_RUS.md)

Reusable GitHub Actions workflow that scans a repository for secrets and IaC misconfigurations with gitleaks, kics and checkov.

The scanners run from signed images in Zot, on the `["self-hosted","homelab"]` runner, without docker. By default they only report: the job stays green until you ask it to fail.

## What it runs

| Tool       | Finds                                         | Gate counts                               |
| ---------- | --------------------------------------------- | ----------------------------------------- |
| `gitleaks` | secrets in the working tree (not in history)  | every finding                             |
| `kics`     | IaC misconfigurations                         | findings at or above `severity-threshold` |
| `checkov`  | IaC misconfigurations                         | every failed check                        |

Secrets are gitleaks only: kics and checkov have secret scanning turned off.

## Quick start

```yaml
name: security
on:
  push: { branches: [main] }
  pull_request:
  workflow_dispatch:

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

The repository needs access to the runner group and the secrets `ZOT_USER` and `ZOT_PASSWORD`. Pass them explicitly, never with `secrets: inherit`. To run only by hand, keep `workflow_dispatch` as the single trigger.

## Inputs

| Input                                                    | Default                      | Meaning                                                          |
| -------------------------------------------------------- | ---------------------------- | ---------------------------------------------------------------- |
| `runner`                                                 | `["self-hosted","homelab"]`  | JSON array of `runs-on` labels                                   |
| `timeout-minutes`                                        | `30`                         | job timeout                                                      |
| `repository`, `ref`                                      | the calling repository       | scan another public repository                                   |
| `artifact-name`                                          | `security-pipeline-reports`  | name of the artifact with the reports                            |
| `severity-threshold`                                     | `HIGH`                       | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` or `INFO`; kics only         |
| `disable-gitleaks`, `disable-kics`, `disable-checkov`    | `false`                      | skip a tool                                                      |
| `raise-failure`                                          | `false`                      | fail the job when a tool errors or has gated findings            |
| `gitleaks-raise-failure`, `kics-raise-failure`, `checkov-raise-failure` | `false`        | the same for one tool                                            |
| `image-gitleaks`, `image-kics`, `image-checkov`          | pinned                       | image pin, see [Image pins](#image-pins)                         |

Secrets `zot-user` and `zot-password` are required: a Zot user that can read `golden/**`.

Outputs: `results` (JSON per tool with `status` `ok`, `error` or `skipped`, `findings`, `gated`, `incomplete`, `raise`) and `verdict` (`pass` or `fail`). Read them in a later job with `fromJSON(needs.<job>.outputs.results)`.

## How the gate works

- A tool is `ok` when the scanner exits with 0 and its report is valid. Anything else is `error`, never "0 findings".
- The job fails only when `raise-failure` (or the switch of that tool) is set and the tool has an `error` or gated findings.
- A broken pin, a failed `cosign verify`, pull or logout always fails the job.
- checkov OSS has no severity, so every failed check counts. Start blocking mode with `kics-raise-failure: true`.
- `incomplete` is how many parts a scanner could not read. It only warns.

## Reports

The artifact is kept for 7 days and has `gitleaks.json`, `kics/kics.json` and `checkov.json`. Reports are rebuilt from a list of allowed fields, so secret values and source text are dropped. File and resource names are still visible, and the artifact of a public repository can be downloaded by any signed-in GitHub user.

## Security

- The workflow fails on `pull_request_target` and `workflow_run`. Fork pull requests get no secrets, so guard the job as in the example.
- Zot credentials exist only while the images are pulled and are removed before the scans.
- A scanner sees the workspace read-only and its own output directory, nothing else.
- `.checkov.yaml` and `kics.config` of the scanned repository are ignored: they can run code or swap the queries. `.gitleaks.toml` is honoured.
- Inline suppressions (`#checkov:skip`, `# kics-scan ignore-line`, `gitleaks:allow`) work, so review them like code.
- Scanning a repository you do not control through `repository` is best effort: the scanners still parse its files.

## Image pins

The defaults are `zot.homelab/golden/<tool>:<date>-<sha>@sha256:<digest>` in `.github/workflows/security-pipeline.yml`. Each one is checked against a regular expression and verified with `common/image/cosign.pub` before the pull. Renovate cannot see Zot, so bump by hand: `skopeo inspect` for the digest, `cosign verify --key common/image/cosign.pub --insecure-ignore-tlog=true <repo>@<digest>`, then a pull request.

## Limitations

- gitleaks scans the working tree only. For the history use [`ci-templates/security.yml`](https://github.com/as-platform/ci-templates), and call one of the two workflows in a repository, not both.
- There is no network isolation, only scanner flags keep them offline.
- Only public repositories can be scanned through `repository`. A monorepo sub directory cannot be scanned alone.
- The Zot user can also write to Zot: connect only repositories you trust.

## Examples

- `examples/lvl_2_base.yml`: soft mode on push, pull request and manual run
- `examples/lvl_5_blocking.yml`: `raise-failure: true` and a deploy job that waits for the gate
- `examples/scan_external_repo.yml`: manual scan of another public repository

## Related repositories

- [`ci-templates`](https://github.com/as-platform/ci-templates): `zot-login` and the service workflows
- [`golden-images`](https://github.com/as-platform/golden-images): builds and signs the scanner images
- [`runner-smoke`](https://github.com/as-platform/runner-smoke): end to end checks of this pipeline
- [`k3s`](https://github.com/as-platform/k3s): GitOps repository with the runner and Zot

## License

MIT, see [`LICENSE`](LICENSE).
