# Пины образов

🇬🇧 [English version](images.md)

По умолчанию заданы пины вида `zot.homelab/golden/<tool>:<YYYYMMDD-sha>@sha256:<digest>` из прогона `golden-images`, который их собрал. Значение проверяется регулярным выражением `^zot\.homelab/golden/(gitleaks|trivy|kics|checkov):[0-9]{8}-[0-9a-f]{7,}@sha256:[0-9a-f]{64}$` до любого pull. В `cosign` и `buildah` уходит только `repo@digest`: skopeo 1.13.3 на раннере отвергает `repo:tag@digest`. `stable` во время запуска не резолвится.

| Вход             | Пин                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `image-gitleaks` | `zot.homelab/golden/gitleaks:20261118-0dcd8e4@sha256:4b2786189367f5bfdc65de189a1586d8060c3d7d7a315ae7fceb5c94d4a8237f`  |
| `image-kics`     | `zot.homelab/golden/kics:20261118-0dcd8e4@sha256:2ad5e4a1b087142af161df599d307da2ef2f18a88c0b3d501c260a8493bf68b1`      |
| `image-checkov`  | `zot.homelab/golden/checkov:20261118-0dcd8e4@sha256:75f39563ee79f00b6a583dcbf2c089020a7ee25c01af11ad606ac313a86d2933`   |

Renovate не видит Zot, поэтому пины обновляются вручную:

1. На хосте с логином в Zot прочитай digest нового датированного тега: `skopeo inspect --format '{{.Digest}}' docker://zot.homelab/golden/<tool>:<YYYYMMDD-sha>`.
2. Проверь его: `cosign verify --key common/image/cosign.pub --insecure-ignore-tlog=true zot.homelab/golden/<tool>@<digest>`.
3. Замени значение по умолчанию `image-<tool>` в `.github/workflows/security-pipeline.yml` и открой pull request.

Пин удалённого тега перестаёт работать, поэтому будущий ретеншен `golden/**` должен сохранять закреплённые теги. Ключ `common/image/cosign.pub` ротируется заменой этого единственного файла.

