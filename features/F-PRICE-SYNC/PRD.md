# F-PRICE-SYNC — выгрузка цены в Яндекс Маркет

Статус: `specified`  
Волна: `W2-CORE`  
Граф: см. `../../graph/graph.yaml` (id: `F-PRICE-SYNC`)  
Трекер: —  
Прогон: [postman-catalog-checklist.md](../../research/briefs/postman-catalog-checklist.md) (2026-09-02)

## Context / Problem

As-is: цена на 861370 из Postman проходит только на кабинет: `POST /v2/businesses/861370/offer-prices/updates` (`value` 945, `discountBase` 1030). Campaign-URL даёт `LOCKED` / `Partner use only default price` (`onlyDefaultPrice`). Плагин `ym` шлёт **campaign** и ключ `"id"` — CLI на этом кабинете упрётся в LOCKED. UI цену не шлёт. Цены на `warehouseId` нет.

To-be: cron CLI ставит цену Shop на все магазины кабинета тем же URL/телом, что Postman.

AS IS: `../../research/as-is/ym-catalog-price-stock-wa.md`.

## Scope

### In

- Смена URL CLI на `businesses/{business_id}/offer-prices/updates`
- В теле `"offerId"` (допустимо дублировать `"id"`)
- Узкий `set_products`; интервал `cli_upload_prices` > 0
- `discountBase` только если проходит правило 5–75%

### Out

- Цена на один склад / один `warehouseId` (API не умеет)
- Уникальные цены по `campaignId`, пока `onlyDefaultPrice` = true
- Карточка (`F-CATALOG`), остаток (`F-STOCK-SYNC`)
- Кнопка UI «в Яндекс.Маркет» для цены (пока нет — не обязательна в P0)
- YMWB

## Preconditions (depends_on)

| ID | Что должно быть правдой |
|----|-------------------------|
| `C-YM-API` | OAuth профиля 17 |
| `F-CATALOG` | Оффер с тем же `offerId` уже в кабинете (на тесте `1107932` уже был) |

## Shared rules / entities

| ID | Как используем |
|----|----------------|
| `E-OFFER` | `offerId` цены = ключ карточки |
| `C-YM-API` | сейчас `apiGetOfferPricesUpdates` → campaign; нужен business |

## Business rules (локальные для фичи)

| ID | Правило | Источник |
|----|---------|----------|
| BR-1 | На 861370 цена только business-URL | Postman LOCKED на campaign |
| BR-2 | Нет поля склада в теле цены | дока updateBusinessPrices + ответ 2026-09-02 |
| BR-3 | `currencyId` = `RUR`; `value` целое | плагин + эталон |
| BR-4 | `discountBase` отбрасывать вне 5–75% к value | `getDataForAddPricesByHash` |
| BR-5 | UI не выгружает цену; автомат = CLI/cron | `shopYmCli` |

## Flows

### Happy path

1. Цена SKU в Shop.
2. `php cli.php shop ym 17` при включённой загрузке цен и узком сете.
3. POST `…/businesses/861370/offer-prices/updates`.
4. Кабинет: цена на всех магазинах (лаг / карантин).

### Fail / edge

| Кейс | Ожидание |
|------|----------|
| Остался campaign-URL | `LOCKED`; не считать успехом |
| `set_products` = all | запрещено на боевом профиле |
| Карантин цены | лог + кабинет; не ретраить вслепую |

## Data / contracts

Эталон: [offer-1107932-prices.json.md](../../research/evidence/external/offer-1107932-prices.json.md).

```json
{
  "offers": [
    {
      "offerId": "1107932",
      "price": { "value": 945, "discountBase": 1030, "currencyId": "RUR" }
    }
  ]
}
```

Код: `apiGetOfferPricesUpdates`, `getApiUrl('/offer-prices/updates')` (сейчас `campaigns=true`).

## Systems touched

| Система | Что меняется |
|---------|--------------|
| WA `plugins/ym` | URL + ключ `offerId` в CLI цен |
| Partner API | `updateBusinessPrices` |
| Кабинет 861370 | цена всех магазинов |

## Acceptance criteria

```gherkin
Given профиль 17, business_id 861370, SKU 1107932 в узком сете
When CLI выгружает цены
Then POST идёт на /v2/businesses/861370/offer-prices/updates
And тело содержит offers[].offerId и price.value из Shop
And нет LOCKED
And query.17.log содержит этот URL
```

Негатив:

```gherkin
Given CLI всё ещё бьёт в campaigns/137514772/offer-prices/updates
When выгрузка цены
Then ответ LOCKED / Partner use only default price
And это fail AC, не «отправлено»
```

```gherkin
When оператор ждёт другую цену только на warehouseId 1669219
Then это вне скоупа: цена кабинета одна
```

## Related / impact

- `shares_entity` `E-OFFER` с `F-CATALOG` / `F-STOCK-SYNC`
- Blast radius: смена URL цен на business затронет кабинеты, где `onlyDefaultPrice` = false (нужен флаг/настройка, не сломать чужой профиль)

## Open questions

| ID | Вопрос |
|----|--------|
| | Нужен ли переключатель campaign vs business по `POST …/businesses/{id}/settings` (`onlyDefaultPrice`) |
| Q-003 | Правило `discountBase` vs 1.18 из старого YMWB |

## Traceability

Прогон 2026-09-02 LOCKED + OK на business → чеклист P0.1–P0.2 → этот PRD → `shopYmApi` / `shopYmCli`.
