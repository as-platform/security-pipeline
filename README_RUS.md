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

### Что находит

- `gitleaks`: секреты в рабочем дереве по стандартному набору правил плюс `.gitleaks.toml` и `.gitleaksignore` проверяемого репозитория. История git не сканируется.
- `kics` и `checkov`: ошибки конфигурации инфраструктуры как кода, например манифесты Kubernetes, Terraform, Dockerfile и workflow GitHub Actions, и другие форматы, которые поддерживают эти инструменты.

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

Чтобы запускать только вручную, оставь `workflow_dispatch` единственным триггером:

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

## Входы

| Вход                     | По умолчанию                | Смысл                                                                                      |
| ------------------------ | --------------------------- | ------------------------------------------------------------------------------------------ |
| `runner`                 | `["self-hosted","homelab"]` | JSON-массив меток `runs-on`                                                                |
| `timeout-minutes`        | `30`                        | таймаут job, держи его меньше `activeDeadlineSeconds` раннера                              |
| `repository`             | `""`                        | `owner/name` публичного репозитория для скана, пусто значит репозиторий вызывающего        |
| `ref`                    | `""`                        | ветка, тег или SHA, пусто значит ref события (ветка по умолчанию для другого репозитория)  |
| `artifact-name`          | `security-pipeline-reports` | имя artifact с отчётами                                                                    |
| `severity-threshold`     | `HIGH`                      | одно из `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`; действует только на kics              |
| `image-gitleaks`         | пин, см. [Пины образов](docs/images_RUS.md) | пин образа                                                                                 |
| `image-kics`             | пин, см. [Пины образов](docs/images_RUS.md) | пин образа                                                                                 |
| `image-checkov`          | пин, см. [Пины образов](docs/images_RUS.md) | пин образа                                                                                 |
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
| `results` | компактный JSON-объект по инструментам: `{"gitleaks":{"status":"ok","findings":4,"gated":4,"incomplete":0,"raise":false},"kics":{...},...}` |
| `verdict` | `fail`, если инструмент сработал на гейте или job упал до гейта, иначе `pass`                                          |

`status` — `ok`, `error` или `skipped`. Выходы читаются в следующем job через `needs.<job>.outputs.results` и `fromJSON`, см. `examples/lvl_5_blocking.yml`.

Что значат `incomplete` и `gated` и когда срабатывает гейт: [Семантика гейта](docs/gate_RUS.md).

## Примеры

| Файл                              | Что показывает                                                                         |
| --------------------------------- | -------------------------------------------------------------------------------------- |
| `examples/lvl_2_base.yml`         | мягкий режим на push, pull request и ручной запуск, с защитой от форков                |
| `examples/lvl_5_blocking.yml`     | `raise-failure: true` и deploy-job, который ждёт гейт и читает `results`               |
| `examples/scan_external_repo.yml` | ручной скан другого публичного репозитория через `repository` и `ref`                  |

Примеры лежат вне `.github/workflows/`, чтобы не запускаться здесь. `examples/scan_external_repo.yml` сканирует репозиторий, который ты не контролируешь: сначала прочитай [Скан репозитория, который ты не контролируешь](docs/security_RUS.md#скан-репозитория-который-ты-не-контролируешь).

## Граница с ci-templates

[`ci-templates/security.yml`](https://github.com/as-platform/ci-templates) проверяет историю git через gitleaks и запускает `trivy fs`, hadolint и semgrep бинарями хоста раннера. Этот workflow запускает подписанные образы, проверяет только рабочее дерево и принимает решение по своим результатам. В репозитории вызывай один из двух, не оба. `trivy config` в этот workflow не входит.

## Ограничения

- gitleaks работает с `--no-git` по checkout глубиной 1: секрет, удалённый из дерева, но оставшийся в истории, не находится. Такой секрет нужно ротировать, а историю, где это важно, проверять через `ci-templates/security.yml`.
- Сетевой изоляции нет. Сканеры работают офлайн только благодаря флагам (`--skip-download` у checkov, запросы kics внутри образа); раннер их трафик не блокирует.
- Через `repository` можно сканировать только публичные репозитории: токена для checkout нет.
- `concurrency` не задан, его решает вызывающий. Пул раннеров мал (`maxReplicaCount 3`), поэтому pipeline на каждый push занимает один под на запуск.
- Подкаталог монорепозитория отдельно сканировать нельзя, проверяется весь checkout.
- Пользователь Zot, который используется сейчас, умеет и писать в Zot. Подключай только репозитории, которым ты это доверяешь, пока нет пользователя только на чтение `golden/**`.

## Документация

| Документ                                        | О чём                                                                           |
| ----------------------------------------------- | ------------------------------------------------------------------------------- |
| [`docs/gate_RUS.md`](docs/gate_RUS.md)           | `raise-failure`, `status`, `gated`, `incomplete`, файлы результатов             |
| [`docs/reports_RUS.md`](docs/reports_RUS.md)     | файлы артефакта и allowlist'ы, которые не пускают значения секретов в отчёты    |
| [`docs/architecture_RUS.md`](docs/architecture_RUS.md) | структура, порядок шагов, песочница сканера, почему один job              |
| [`docs/images_RUS.md`](docs/images_RUS.md)       | пины образов и как их обновлять                                                 |
| [`docs/security_RUS.md`](docs/security_RUS.md)   | подавления, заметки по безопасности, скан чужого репозитория                    |

## Связанные репозитории

- [`ci-templates`](https://github.com/as-platform/ci-templates) — `actions/zot-login` и workflow, рядом с которыми стоит этот pipeline
- [`golden-images`](https://github.com/as-platform/golden-images) — собирает и подписывает образы сканеров
- [`runner-smoke`](https://github.com/as-platform/runner-smoke) — гоняет сквозные проверки pipeline на раннере
- [`k3s`](https://github.com/as-platform/k3s) — GitOps-репозиторий с раннером и Zot

## Лицензия

MIT — см. [`LICENSE`](LICENSE).
