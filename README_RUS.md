# security-pipeline

🇬🇧 [English version](README.md)

Reusable workflow, который проверяет репозиторий на секреты и ошибки конфигурации IaC через gitleaks, kics и checkov. Сканеры запускаются из подписанных образов в реестре Zot домашней платформы [as-platform](https://github.com/as-platform) на эфемерном раннере `["self-hosted","homelab"]`, без docker: образы проверяются `cosign`, скачиваются `buildah from` и запускаются `buildah run`.

Сканеры всегда работают в режиме отчёта. Один шаг на bash и `jq` читает их результаты и решает, падает ли job, поэтому скан не ломает pipeline, пока вызывающий сам этого не попросит.

## Что запускается

| Инструмент | Практика       | Режим                                                         | `gated` это                                  |
| ---------- | -------------- | ------------------------------------------------------------- | -------------------------------------------- |
| `gitleaks` | `SAST/SECRETS` | `detect --no-git --redact`, только рабочее дерево             | каждая находка                               |
| `kics`     | `SAST/IAC`     | `scan --ignore-on-exit results --disable-secrets`             | находки не ниже `severity-threshold`         |
| `checkov`  | `SAST/IAC`     | `--soft-fail --skip-framework secrets --skip-download`        | каждая проваленная проверка                  |

Секреты ищет только gitleaks: в kics и checkov поиск секретов выключен, поэтому найденный секрет не печатают три инструмента.

## Быстрый старт

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

Вызывающему репозиторию нужен доступ к раннеру (runner group организации) и секреты `ZOT_USER` и `ZOT_PASSWORD`. Секреты передаются явно, `secrets: inherit` не используется. Workflow подключается по `@main`, как и все остальные шаблоны платформы.

## Входы

| Вход                     | По умолчанию                | Смысл                                                                                      |
| ------------------------ | --------------------------- | ------------------------------------------------------------------------------------------ |
| `runner`                 | `["self-hosted","homelab"]` | JSON-массив меток `runs-on`                                                                |
| `timeout-minutes`        | `30`                        | таймаут job, держи его меньше `activeDeadlineSeconds` раннера                              |
| `repository`             | `""`                        | `owner/name` публичного репозитория для скана, пусто значит репозиторий вызывающего        |
| `ref`                    | `""`                        | ветка, тег или SHA, пусто значит ref события (ветка по умолчанию для другого репозитория)  |
| `artifact-name`          | `security-pipeline-reports` | имя artifact с отчётами                                                                    |
| `severity-threshold`     | `HIGH`                      | одно из `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`; действует только на kics              |
| `image-gitleaks`         | пин, см. ниже               | пин образа                                                                                 |
| `image-kics`             | пин, см. ниже               | пин образа                                                                                 |
| `image-checkov`          | пин, см. ниже               | пин образа                                                                                 |
| `disable-gitleaks`       | `false`                     | пропустить инструмент                                                                      |
| `disable-kics`           | `false`                     | пропустить инструмент                                                                      |
| `disable-checkov`        | `false`                     | пропустить инструмент                                                                      |
| `raise-failure`          | `false`                     | уронить job, если любой включённый инструмент в ошибке или имеет gated-находки             |
| `gitleaks-raise-failure` | `false`                     | то же только для gitleaks                                                                  |
| `kics-raise-failure`     | `false`                     | то же только для kics                                                                      |
| `checkov-raise-failure`  | `false`                     | то же только для checkov                                                                   |

Групповых переключателей вроде `disable-sast` или `disable-iac` нет, только поинструментные. Реестр `zot.homelab` зашит, входа `registry` нет.

## Секреты

| Секрет         | Смысл                                  |
| -------------- | -------------------------------------- |
| `zot-user`     | пользователь Zot с чтением `golden/**` |
| `zot-password` | пароль этого пользователя              |

Оба обязательны. Пустое значение валит шаг `prepare` сообщением про fork pull request: GitHub не передаёт им секреты, и pipeline падает, а не пропускает скан молча (fail-closed).

## Выходы

| Выход     | Смысл                                                                                                                  |
| --------- | ---------------------------------------------------------------------------------------------------------------------- |
| `results` | компактный JSON-объект по инструментам: `{"gitleaks":{"status":"ok","findings":4,"gated":4,"raise":false},"kics":{...},...}` |
| `verdict` | `fail`, если инструмент сработал на гейте или упала фаза до сканов, иначе `pass`                                       |

`status` — `ok`, `error` или `skipped`. Выходы читаются в следующем job через `needs.<job>.outputs.results` и `fromJSON`, см. `examples/lvl_5_blocking.yml`.

## Семантика гейта

- Инструмент `ok`, когда сканер завершился с кодом 0 и его отчёт — валидный JSON ожидаемой формы. Ненулевой код, отсутствующий или невалидный отчёт, либо сканер, который не запускался, — это `error`, а не «0 находок». Выключенный инструмент — `skipped`.
- Гейт срабатывает на инструменте, когда `raise-failure` или `<tool>-raise-failure` равен `true`, а инструмент в `error` или имеет `gated > 0`.
- По умолчанию не срабатывает ничего: job зелёный, `error` виден аннотацией `::error::`, а таблица в job summary показывает `tool | practice | status | findings | gated (threshold) | raise`.
- Job падает независимо от переключателей, если упал `prepare`, проверка пина, `cosign verify`, pull или logout. Сбой целостности или инфраструктуры никогда не бывает мягким.
- `gated` у kics считает severity от `severity-threshold` и выше. checkov OSS не отдаёт severity, поэтому порог на него не действует и gated каждая проваленная проверка: он срабатывает по принципу «всё или ничего». При включении блокировки начни с `kics-raise-failure: true`.
- Шаг `Quality gate` сначала пишет summary и выходы, а падает job на следующем шаге `Enforce the gate`, поэтому выходы заданы и тогда, когда гейт сработал.

Каждый инструмент пишет `$RUNNER_TEMP/security-pipeline/results/<tool>.json`:

```json
{"tool": "kics", "practice": "SAST/IAC", "status": "ok", "rc": 0, "findings": 290, "gated": 56, "report": "kics/kics.json"}
```

`rc` равен `-1`, если сканер не запускался. Отсутствующий файл результата считается `error`.

## Отчёты

Artifact `artifact-name` (хранение 7 дней, пустой набор не ошибка) содержит `gitleaks.json`, `kics/kics.json` и `checkov.json`. Значений секретов в отчётах нет: gitleaks работает с `--redact` и без `-v`, поэтому каждый `Secret` равен `REDACTED`, а значение внутри `Match` заменено на `REDACTED`; kics запускается с `--disable-secrets`, а его вывод в консоль уходит в файл, который не загружается; checkov пропускает framework `secrets`, а из его отчёта вырезаются `code_block` и `check_result.entity`, потому что checkov копирует туда атрибуты ресурсов (пароли, user data) и без framework `secrets`.

## Примеры

| Файл                              | Что показывает                                                                         |
| --------------------------------- | -------------------------------------------------------------------------------------- |
| `examples/lvl_2_base.yml`         | мягкий режим на push, pull request и ручной запуск, с защитой от форков                |
| `examples/lvl_5_blocking.yml`     | `raise-failure: true` и deploy-job, который ждёт гейт и читает `results`               |
| `examples/scan_external_repo.yml` | ручной скан другого публичного репозитория через `repository` и `ref`                  |

Примеры лежат вне `.github/workflows/`, чтобы не запускаться здесь.

## Структура

| Путь                                      | Назначение                                                        |
| ----------------------------------------- | ----------------------------------------------------------------- |
| `.github/workflows/security-pipeline.yml` | reusable workflow, один job `security-pipeline`                   |
| `common/prepare`                          | проверяет входы, инструменты хоста и секреты, создаёт рабочий каталог |
| `common/image`                            | проверка пина, `cosign verify` по digest, `buildah from`          |
| `common/image/cosign.pub`                 | публичный ключ, которым проверяются образы `golden-images`        |
| `common/logout`                           | убирает все креды Zot до сканов                                   |
| `common/quality_gate`                     | читает результаты, пишет summary и выходы                         |
| `tools/SAST/SECRETS/gitleaks`             | скан gitleaks                                                     |
| `tools/SAST/IAC/{kics,checkov}`           | сканы kics и checkov                                              |
| `examples`                                | вызывающие workflow по уровням зрелости                           |

Шаги job идут в таком порядке: `checkout`, `prepare`, `zot-login`, `image` для каждого включённого инструмента, `logout`, сканы по очереди, `quality_gate`, загрузка отчётов, удаление контейнеров. Сканы идут с `if: !cancelled()`, поэтому упавший инструмент не скрывает остальные.

Сканер запускается как `buildah run --user 0:0`, workspace примонтирован только на чтение (`:ro`), а `$RUNNER_TEMP/security-pipeline` примонтирован на запись для отчётов. `--user 0:0` — это root rootless user namespace, то есть непривилегированный пользователь раннера на хосте. Целиком `$RUNNER_TEMP` (там `GITHUB_ENV` и `GITHUB_PATH`), `$HOME` и authfile не монтируются. Если окажется, что инструменту нужно писать в дерево, на запись монтируется только его контейнер, и этот абзац перечислит такие случаи.

## Почему один job

Каждый job — отдельный под раннера через KEDA, а ёмкость раннера мала. Один job означает один под, один checkout и один `zot-login`. Если нужен параллелизм, вызови workflow дважды с дополняющими друг друга `disable-*` и разными `artifact-name`.

## Пины образов

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

## Граница с ci-templates

[`ci-templates/security.yml`](https://github.com/as-platform/ci-templates) проверяет историю git через gitleaks и запускает `trivy fs`, hadolint и semgrep бинарями хоста раннера. Этот workflow запускает подписанные образы, проверяет только рабочее дерево и принимает решение по своим результатам. В репозитории вызывай один из двух, не оба. `trivy config` в этот workflow не входит.

## Ограничения

- gitleaks работает с `--no-git` по checkout глубиной 1: секрет, удалённый из дерева, но оставшийся в истории, не находится. Такой секрет нужно ротировать, а историю, где это важно, проверять через `ci-templates/security.yml`.
- Сетевой изоляции нет. Сканеры работают офлайн только благодаря флагам (`--skip-download` у checkov, запросы kics внутри образа); раннер их трафик не блокирует.
- Через `repository` можно сканировать только публичные репозитории: токена для checkout нет.
- `concurrency` не задан, его решает вызывающий. Пул раннеров мал (`maxReplicaCount 3`), поэтому pipeline на каждый push занимает один под на запуск.
- Подкаталог монорепозитория отдельно сканировать нельзя, проверяется весь checkout.
- Пользователь Zot, который используется сейчас, умеет и писать в Zot. Подключай только репозитории, которым ты это доверяешь, пока нет пользователя только на чтение `golden/**`.

## Подавления

Подавления лежат в проверяемом репозитории, поэтому тот, кто может менять код, может заглушить скан. Проверяй их как код и закрой их файлы через `CODEOWNERS`:

- checkov: `#checkov:skip=<id>:<причина>` у ресурса или `.checkov.yaml`
- kics: комментарии `# kics-scan ignore-line` и `# kics-scan ignore-block`
- gitleaks: `.gitleaks.toml`, `.gitleaksignore` и комментарий `gitleaks:allow`

## Заметки по безопасности

- Не вызывай workflow из триггеров `pull_request_target` и `workflow_run`, которые забирают код pull request: у job в первой фазе есть креды Zot, а сканеры работают по забранному дереву.
- Fork pull request не получают секреты, поэтому защити job условием `github.event.pull_request.head.repo.fork != true`, как в примерах.
- Креды Zot живут только в фазе fetch. `common/logout` выполняет `buildah logout --all`, удаляет `~/.docker/config.json` и проверяет, что оба исчезли, до запуска первого сканера. Сторонние actions работают до логина или после logout.
- У workflow `permissions: contents: read`, checkout не сохраняет свой токен.

## Как проверяется репозиторий

Собственного CI у репозитория нет. Локально: `actionlint -config-file .github/actionlint.yaml` по workflow и примерам, `shellcheck -s bash` по каждому `run:` каждого action, grep-ворота на запрещённые конструкции (`${{` внутри `run:`, `ubuntu-latest`, `docker`, `container:`, Python, `secrets: inherit`, внешние actions без SHA) и `npx renovate-config-validator --strict renovate.json`. Поведение проверяет сквозной workflow в `runner-smoke`: он вызывает `security-pipeline@main` на раннере против закреплённой копии `bridgecrewio/terragoat` и против самого `runner-smoke`.

## Связанные репозитории

- [`ci-templates`](https://github.com/as-platform/ci-templates) — `actions/zot-login` и workflow, рядом с которыми стоит этот pipeline
- [`golden-images`](https://github.com/as-platform/golden-images) — собирает и подписывает образы сканеров
- [`runner-smoke`](https://github.com/as-platform/runner-smoke) — гоняет сквозные проверки pipeline на раннере
- [`k3s`](https://github.com/as-platform/k3s) — GitOps-репозиторий с раннером и Zot

## Лицензия

MIT — см. [`LICENSE`](LICENSE).
