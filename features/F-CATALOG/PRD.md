# F-CATALOG — выгрузка карточки оффера в Яндекс Маркет

Статус: `specified`  
Волна: `W1-FOUNDATION`  
Граф: см. `../../graph/graph.yaml` (id: `F-CATALOG`)  
Трекер: —  
Прогон: [postman-catalog-checklist.md](../../research/briefs/postman-catalog-checklist.md) (2026-09-02, SKU `1107932`)

## Context / Problem

As-is: Partner API карточки на кабинете **861370** из Postman работает (`POST …/offer-mappings/update`, `"status": "OK"`, кабинет обновился). Плагин `ym` уже умеет тот же URL, но UI часто «успех без API» (нет hash коллекции), CLI шлёт устаревший ключ `offerMappingEntries`, поля тела (`shopSku`, `mapping` внутри `offer`, `customsCommodityCodes`) не совпадают с эталоном прогона.

To-be: cron/CLI (и починенный UI) выгружает карточку Shop → Маркет так же, как эталон Postman, на узком `shop_set`, без ручного Postman.

Reality Brief: `../../research/briefs/reality-brief.md`. AS IS: `../../research/as-is/ym-catalog-price-stock-wa.md`.  
`D-YM-CARD-GEN` остаётся **proposed** (генератор категорий как Ozon/WB). Этот PRD — **P0 патч существующего `ym`**, не новый контент-хаб. Fast Lane после прогона 2026-09-02.

## Scope

### In

- `business_id` профиля = кабинет **861370** (не campaign)
- `offerId` / identification **0** = `shop_product_skus.id` (на тесте `1107932`)
- Тело UI: `offerMappings`; CLI — тот же ключ, что принимает Маркет (`offerMappings`; `offerMappingEntries` только как fallback при 400)
- Поля как эталон: `offerId`, `mapping` сосед `offer`, `customsCommodityCode` (строка), vendor/barcode/`vendorCode`
- Починка hash: галка строки не должна давать 100% без вызова API
- Узкий список `set_products` для CLI (не `all`)

### Out

- Цены (`F-PRICE-SYNC`), остатки (`F-STOCK-SYNC`)
- Генератор категорий `marketCategoryId` / `parameterValues` из `D-YM-CARD-GEN` A
- YMWB / `offerId` = `Sklad`
- Массовая выгрузка всего каталога
- Заказы (`F-ORDERS`)

## Preconditions (depends_on)

| ID | Что должно быть правдой |
|----|-------------------------|
| `C-YM-API` | OAuth профиля 17 ходит в Partner API (прогон 2026-09-02 = HTTP 200) |
| `D-YM-CARD-GEN` | Не блокер P0: карточка уже уходит `offer-mappings/update`. Полный маппинг категорий — отдельный принятый D |

## Shared rules / entities

| ID | Как используем |
|----|----------------|
| `E-OFFER` | Ключ оффера = `offerId` = `shopSku` на профиле 17 (`sku_id`) |
| `C-YM-API` | `shopYmApi::apiOfferMappingEntriesUpdates` → business URL |

## Business rules (локальные для фичи)

| ID | Правило | Источник |
|----|---------|----------|
| BR-1 | Карточка только `POST /v2/businesses/{businessId}/offer-mappings/update` | Postman 2026-09-02; `shopYmApi` |
| BR-2 | `businessId` ≠ `campaignId`. Стенд: 861370 ≠ 137514772 | `Incorrect businessId: offer-mappings` |
| BR-3 | Пустая коллекция ≠ успех: не писать «Отправлено без ошибок» без запроса | баг hash UI |
| BR-4 | Эталон `vendorCode` = `JB-22282-1107932` (не только штрихкод) | тело прогона |
| BR-5 | `mapping.marketSku` — сосед `offer`, не внутри | эталон + дока updateOfferMappings |

## Flows

### Happy path

1. SKU в узком сете профиля 17.
2. CLI `php cli.php shop ym 17` (интервал карточек > 0) **или** список `#/id/{product_id}/` + «выбрать все» + «в Яндекс.Маркет».
3. POST business `offer-mappings/update` с `offerId`.
4. `query.17.log`: URL кабинета, тело, `"status": "OK"`.
5. Кабинет: поля как в эталоне (лаг минуты).

### Fail / edge

| Кейс | Ожидание |
|------|----------|
| `business_id` пустой | нет тихого 100%; ошибка в UI/`error.17.log` |
| Галка только на строке каталога | не слать пустой батч как успех |
| SKU нет в Shop (заглушка `2`) | не выгружать |

## Data / contracts

Эталон тела: [offer-1107932-mappings.json.md](../../research/evidence/external/offer-1107932-mappings.json.md) + чеклист §1.

Код: `getDataForAddOfferByHash`, `shopYmPluginBackendSendToYm`, `shopYmCli`.

## Systems touched

| Система | Что меняется |
|---------|--------------|
| WA `plugins/ym` | URL/тело/hash UI; профиль `business_id` |
| Partner API | `offer-mappings/update` |
| Кабинет 861370 | карточка оффера |

## Acceptance criteria

```gherkin
Given профиль 17, business_id 861370, identification 0, SKU 1107932 в узком сете
When CLI (или починенный UI) выгружает карточку
Then POST идёт на /v2/businesses/861370/offer-mappings/update
And тело содержит offerMappings[].offer.offerId = "1107932"
And query.17.log фиксирует запрос и OK
And в кабинете имя/vendor/barcode/ТН ВЭД совпадают с Shop (лаг допустим)
```

Негатив:

```gherkin
Given галка только на строке общего списка без hash
When оператор жмёт «в Яндекс.Маркет»
Then нет отчёта «без ошибок» без вызова Partner API
```

```gherkin
Given business_id не 861370
When выгрузка карточки
Then запрос не маскируется под успех (ожидаем BAD_REQUEST / лог ошибки)
```

## Related / impact

- Далее: `F-PRICE-SYNC`, `F-STOCK-SYNC` (тот же `offerId`)
- Blast radius: смена identification ломает заказы и остатки
- `D-YM-CARD-GEN` A (`offerId` = Sklad) **конфликтует** с прогоном (`sku_id`); не принимать A без новой D

## Open questions

| ID | Вопрос |
|----|--------|
| Q-009 | Какой постоянный `shop_set` вместо тестового |
| Q-010 | Нужен ли сразу `marketCategoryId` / parameterValues |
| P1.11 | Формула `vendorCode` в `getVendorCodeByProductAndSku` |

## Traceability

Прогон 2026-09-02 → чеклист P0.3/P0.7 + P1.8–12 → этот PRD → патч `shopYmApi` / sendToYm / Cli.
