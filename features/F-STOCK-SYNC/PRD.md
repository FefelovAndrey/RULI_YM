# F-STOCK-SYNC — выгрузка остатков в Яндекс Маркет

Статус: `specified`  
Волна: `W2-CORE`  
Граф: см. `../../graph/graph.yaml` (id: `F-STOCK-SYNC`)  
Трекер: —  
Прогон: [postman-catalog-checklist.md](../../research/briefs/postman-catalog-checklist.md) (2026-09-02)

## Context / Problem

As-is: остаток на складе **1669219** (кампания **137514772**, «Ruli МСК 1 день») из Postman проходит `PUT /v2/campaigns/{campaignId}/offers/stocks` с массивом `skus`, `type: FIT`. Если `updatedAt` старше уже лежащей записи — HTTP OK, но **меняется только дата, не `count`**. Один запрос = один `warehouseId`. В кабинете 18 складов, `warehouseGroups: []`. Плагин шлёт тот же PUT, но одно поле `warehouse_id` на профиль и `date('c')` сервера WA — при рассинхроне часов count не едет. UI остаток не шлёт.

To-be: CLI выставляет count Shop на нужные склады (цикл `campaignId` ↔ `warehouseId`) со **свежим** `updatedAt` в TZ Маркета.

AS IS: `../../research/as-is/ym-catalog-price-stock-wa.md`. Склады: [warehouses-861370.md](../../research/evidence/external/warehouses-861370.md).

## Scope

### In

- PUT campaign `offers/stocks` как эталон (не путать с v3 `skuItems` на этом URL)
- `updatedAt` = now, TZ согласован с ЯМ (NTP/TZ `test.ruli.ru`)
- Маппинг складов профиля: не только 1669219, если нужна вся витрина
- Узкий сет; не слать `count: 0` на каталог
- Пороги `min_stock` / `min_price` — явное поведение (0), не сюрприз

### Out

- Цена на склад (`F-PRICE-SYNC`)
- Карточка (`F-CATALOG`)
- Обнуление всех 18 складов «для теста»
- Группы складов (на 861370 их нет)
- YMWB

## Preconditions (depends_on)

| ID | Что должно быть правдой |
|----|-------------------------|
| `C-YM-API` | OAuth профиля 17 |
| `F-CATALOG` | Оффер есть в кабинете (тест: `1107932`) |

## Shared rules / entities

| ID | Как используем |
|----|----------------|
| `E-STOCK` | Остаток = `(sku, warehouseId, count)`; sku = `offerId` карточки |
| `E-OFFER` | Тот же ключ `1107932` |
| `C-YM-API` | `setOffersStocks` / `convertPricesToStocks` |

## Business rules (локальные для фичи)

| ID | Правило | Источник |
|----|---------|----------|
| BR-1 | Запись: PUT `…/campaigns/{campaignId}/offers/stocks`, ключ `skus` | Postman 2026-09-02 |
| BR-2 | `updatedAt` не в прошлом относительно записи Маркета, иначе count не меняется | прогон + чеклист §3 |
| BR-3 | Один `warehouseId` на элемент; все склады — отдельные PUT (свой campaign) | GET warehouses 861370 |
| BR-4 | Профиль 17 без цикла закрывает только 1669219 / 137514772 | таблица складов |
| BR-5 | `skuItems` на v2 campaign → `skus size … 0` | Postman |
| BR-6 | Ниже `min_stock` / `min_price` уходит count 0 | плагин |
| BR-7 | Автомат = CLI, не UI | `shopYmCli` |

## Flows

### Happy path

1. Остаток SKU в Shop на складе, который мапится на 1669219.
2. CLI с включённой загрузкой остатков, узкий сет, свежий last_time.
3. PUT campaign 137514772, `warehouseId` 1669219, `count` из Shop, `updatedAt` = now.
4. Кабинет на «Ruli МСК 1 день»: то же число (лаг до ~15 мин).

### Fail / edge

| Кейс | Ожидание |
|------|----------|
| Старый `updatedAt` | OK, count прежний — **fail** приёмки |
| Тело `skuItems` на campaign PUT | BAD_REQUEST size 0 |
| CLI на `all` | запрещено |
| Смотрим ЕКБ / публичный market.ru | не критерий приёмки |

## Data / contracts

Эталон: [offer-1107932-stocks.json.md](../../research/evidence/external/offer-1107932-stocks.json.md).  
`updatedAt` в примере не копировать — всегда «сейчас».

Код: `convertPricesToStocks` (`date('c')`), `setOffersStocks`.

Тест склада: **1669219** / campaign **137514772**. Остальные id — только по явному списку, не «все 18 сразу» на первом прогоне CLI.

## Systems touched

| Система | Что меняется |
|---------|--------------|
| WA `plugins/ym` | TZ/`updatedAt`; маппинг складов; CLI цикл |
| Partner API | PUT `offers/stocks` |
| Кабинет, склад 1669219 | count |

## Acceptance criteria

```gherkin
Given профиль 17, SKU 1107932, склад ЯМ 1669219, часы сервера = TZ ЯМ
When CLI выгружает остатки
Then PUT идёт на /v2/campaigns/137514772/offers/stocks
And тело skus[0].sku = "1107932", warehouseId = 1669219, items[0].type = FIT
And updatedAt не старше текущей записи Маркета
And в кабинете на этом складе count совпадает (лаг до 15 мин)
And query.17.log содержит этот PUT
```

Негатив:

```gherkin
Given updatedAt в прошлом относительно стока Маркета
When PUT с новым count
Then HTTP может быть OK
And count в кабинете не меняется — это fail, пока CLI не шлёт now
```

```gherkin
Given CLI с set_products = all
When запуск shop ym 17
Then выгрузка остатков не стартует (защита)
```

## Related / impact

- `F-ORDERS` зависит от живого оффера на витрине, не от полного цикла 18 складов
- Blast radius: цикл по всем складам без маппинга Shop→ЯМ обнулит чужие витрины

## Open questions

| ID | Вопрос |
|----|--------|
| Q-007 | Нужны ли все 18 складов в первом релизе или только 1669219 |
| | Источник count Shop: какой `stock_id` → 1669219 |

## Traceability

Прогон 2026-09-02 (OK + «дата без count») → чеклист P0.4–P0.6 → этот PRD → `convertPricesToStocks` / CLI.
