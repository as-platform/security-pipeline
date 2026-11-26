# Семантика гейта

🇬🇧 [English version](gate.md)

- Инструмент `ok`, когда сканер завершился с кодом 0 и его отчёт — валидный JSON ожидаемой формы. Ненулевой код, отсутствующий или невалидный отчёт, либо сканер, который не запускался, — это `error`, а не «0 находок». Выключенный инструмент — `skipped`.
- Ожидаемая форма checkov — JSON, где у каждого элемента (объект для одного framework, массив для нескольких) есть `summary` с числовыми `failed` и `parsing_errors`; дерево без IaC печатает этот summary как весь объект, и это допустимо. `{}`, `[]`, элемент без summary и любая другая форма — `error`.
- Гейт срабатывает на инструменте, когда `raise-failure` или `<tool>-raise-failure` равен `true`, а инструмент в `error` или имеет `gated > 0`.
- По умолчанию не срабатывает ничего: job зелёный, `error` виден аннотацией `::error::`, а таблица в job summary показывает `tool | practice | status | findings | gated (threshold) | raise`.
- Job падает независимо от переключателей, если упал `prepare` (включая проверку события, см. [Заметки по безопасности](security_RUS.md#заметки-по-безопасности)), проверка пина, `cosign verify`, pull или logout. Сбой целостности или инфраструктуры никогда не бывает мягким.
- Вердикт `fail` и тогда, когда статус job на гейте не `success`, например упал или аварийно завершился шаг до гейта: гейт печатает `the job failed before the gate`.
- `gated` у kics считает severity от `severity-threshold` и выше. checkov OSS не отдаёт severity, поэтому порог на него не действует и gated каждая проваленная проверка: он срабатывает по принципу «всё или ничего». При включении блокировки начни с `kics-raise-failure: true`.
- Шаг `Quality gate` сначала пишет summary и выходы, а падает job на следующем шаге `Enforce the gate`, поэтому выходы заданы и тогда, когда гейт сработал.

Каждый инструмент пишет `$RUNNER_TEMP/security-pipeline/results/<tool>.json`:

```json
{"tool": "kics", "practice": "SAST/IAC", "status": "ok", "rc": 0, "findings": 290, "gated": 56, "incomplete": 0, "report": "kics/kics.json"}
```

`rc` равен `-1`, если сканер не запускался. Отсутствующий файл результата или файл без числового `incomplete` считается `error`. Файл пишет шаг на хосте, вне монтирований любого контейнера.

## Счётчик `incomplete`

`incomplete` — число: сколько частей скана инструмент не смог завершить. У checkov это сумма `summary.parsing_errors`, у kics `files_failed_to_scan + queries_failed_to_execute`, у gitleaks всегда `0`; инструмент в `error` или `skipped` отдаёт `0`. Значение больше 0 никогда не меняет `status`, `gated` и вердикт: инструмент печатает `::warning::<tool> scan incomplete: N` и остаётся `ok`, поэтому чистый результат при `incomplete > 0` означает «в прочитанном ничего не найдено». Замерено на `bridgecrewio/terragoat@729f8da62c6a85ce4af5ad3d123de97776d954c4` и на этом репозитории: `incomplete` равен 0 у всех трёх инструментов.
