# Image pins

🇷🇺 [Русская версия](images_RUS.md)

The defaults are pins of the form `zot.homelab/golden/<tool>:<YYYYMMDD-sha>@sha256:<digest>`, taken from the run of `golden-images` that built them. The value is checked with the regular expression `^zot\.homelab/golden/(gitleaks|trivy|kics|checkov):[0-9]{8}-[0-9a-f]{7,}@sha256:[0-9a-f]{64}$` before anything is pulled. Only `repo@digest` goes to `cosign` and `buildah`: skopeo 1.13.3 on the runner rejects `repo:tag@digest`. `stable` is never resolved at run time.

| Input            | Pin                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `image-gitleaks` | `zot.homelab/golden/gitleaks:20261118-0dcd8e4@sha256:4b2786189367f5bfdc65de189a1586d8060c3d7d7a315ae7fceb5c94d4a8237f`  |
| `image-kics`     | `zot.homelab/golden/kics:20261118-0dcd8e4@sha256:2ad5e4a1b087142af161df599d307da2ef2f18a88c0b3d501c260a8493bf68b1`      |
| `image-checkov`  | `zot.homelab/golden/checkov:20261118-0dcd8e4@sha256:75f39563ee79f00b6a583dcbf2c089020a7ee25c01af11ad606ac313a86d2933`   |

Renovate cannot reach Zot, so the pins are bumped by hand:

1. On a host that is logged in to Zot, read the digest of the new dated tag: `skopeo inspect --format '{{.Digest}}' docker://zot.homelab/golden/<tool>:<YYYYMMDD-sha>`.
2. Verify it: `cosign verify --key common/image/cosign.pub --insecure-ignore-tlog=true zot.homelab/golden/<tool>@<digest>`.
3. Replace the default of `image-<tool>` in `.github/workflows/security-pipeline.yml` and open a pull request.

The pin of a removed tag stops working, so a future retention policy for `golden/**` must keep the pinned tags. The key in `common/image/cosign.pub` is rotated by replacing that single file.

