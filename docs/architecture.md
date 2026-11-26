# Architecture

🇷🇺 [Русская версия](architecture_RUS.md)

## Layout

| Path                                  | Purpose                                                          |
| ------------------------------------- | ---------------------------------------------------------------- |
| `.github/workflows/security-pipeline.yml` | the reusable workflow, one job `security-pipeline`           |
| `common/prepare`                      | checks inputs, host tools and secrets, creates the work directory |
| `common/image`                        | pin check, `cosign verify` by digest, `buildah from`             |
| `common/image/cosign.pub`             | public key that verifies the images of `golden-images`           |
| `common/logout`                       | removes every Zot credential before the scans                    |
| `common/quality_gate`                 | reads the results, writes the summary and the outputs            |
| `tools/SAST/SECRETS/gitleaks`         | gitleaks scan                                                    |
| `tools/SAST/IAC/{kics,checkov}`       | kics and checkov scans                                           |
| `examples`                            | callers by maturity level                                        |

### Step order

1. `checkout`
2. `prepare`
3. `zot-login`
4. `image` for every enabled tool
5. `logout`
6. the scans, one after another
7. `quality_gate`
8. `Enforce the gate`
9. upload of the reports
10. removal of the containers

- The scans run with `if: !cancelled()` and need a successful `logout`, so one failed tool does not hide the others.
- The upload also needs a successful `logout` (`if: always() && steps.logout.outcome == 'success'`), so the third-party upload action never runs while Zot credentials may still be on the runner.

### Scanner sandbox

Each scanner runs as `buildah run --user 0:0` with exactly two mounts:

| Mount                                                | Mode       |
| ---------------------------------------------------- | ---------- |
| the workspace (the checkout)                         | read-only  |
| `$RUNNER_TEMP/security-pipeline/out/<tool>`          | read-write |

Never mounted: `$RUNNER_TEMP/security-pipeline/results` (the input of the gate), `$RUNNER_TEMP/security-pipeline/reports` (the uploaded files), `$RUNNER_TEMP` as a whole (it holds `GITHUB_ENV` and `GITHUB_PATH`), `$HOME` and the auth file. A scanner therefore cannot rewrite its own result, the result of another tool or a report.

After a scan the host step reads the raw output from `out/<tool>`, computes the counters, writes `results/<tool>.json` and the sanitized report into `reports/`, and deletes `out/<tool>`.

- checkov prints its JSON to stdout, and the host step redirects it into a file that no container sees. For checkov `out/checkov` is only the working directory.
- `--user 0:0` is root of the rootless user namespace, that is the unprivileged runner user on the host.
- If a tool turns out to need a writable tree, only its mount becomes read-write and this section lists such cases.

## Why one job

Every job is a separate runner pod started through KEDA, and the runner capacity is small. One job means one pod, one checkout and one `zot-login`. If you need parallelism, call the workflow twice with complementary `disable-*` inputs and different `artifact-name` values.

