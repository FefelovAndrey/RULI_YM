# Чеклист Postman: карточка / цена / остаток

Стенд: кабинет **861370**, кампания теста **137514772**, склад теста **1669219** («Ruli МСК 1 день»).  
SKU: **`1107932`**. Профиль WA: **17**.  
Прогон: 2026-09-02. Канон контура: плагин `ym`, не YMWB.

Auth (не копировать значения в git):

```http
Authorization: OAuth oauth_token="<профиль 17>", oauth_client_id="<client_id>"
Content-Type: application/json
```

Цена **не** ставится на `warehouseId`. Остаток — **да**, по каждому складу отдельно.

Связанные тела: [карточка](../evidence/external/offer-1107932-mappings.json.md), [цена](../evidence/external/offer-1107932-prices.json.md), [остаток](../evidence/external/offer-1107932-stocks.json.md), [склады](../evidence/external/warehouses-861370.md).  
Протокол UI/CLI: [catalog-price-stock-test.md](catalog-price-stock-test.md).

---

## 0. Подготовка

- [ ] OAuth профиля 17, не Basic Auth, не чужой профиль.
- [ ] В Postman три **разных** запроса: не один URL на всё.
- [ ] Снимок кабинета **до**: карточка / цена / остаток на складе 1669219.
- [ ] Не слать `count: 0` и не крутить CLI на `all`.
- [ ] Не трогать заказ `61089913793` / SHIPPED.

---

## 1. Карточка — пройден

| | |
|--|--|
| Method | **POST** |
| URL | `https://api.partner.market.yandex.ru/v2/businesses/861370/offer-mappings/update` |

- [ ] HTTP **200**, `"status": "OK"`.
- [ ] В кабинете через несколько минут: имя, vendor **STONE**, `vendorCode` **JB-22282-1107932**, штрихкод **JB-22282**, ТН ВЭД **8484900000**, фото, габариты, `marketSku` **103757217881**.

Тело (эталон прогона):

```json
{
  "offerMappings": [
    {
      "offer": {
        "offerId": "1107932",
        "name": "Прокладка впускного коллектора MAZDA FAMILIA/323 ZL 99- STONE JB-22282",
        "vendor": "STONE",
        "vendorCode": "JB-22282-1107932",
        "barcodes": ["JB-22282"],
        "description": "Прокладка впускного коллектора MAZDA FAMILIA/323 ZL 99- производства STONE артикул JB-22282.Прокладка впускного коллектора MAZDA FAMILIA/323 ZL 99- устанавливается на Каталог годов выпуска. Характеристики:Артикул: JB-22282 Производитель: STONE Страна производства: Япония Состояние товара: НОВЫЙ (в упаковке)Гарантийный срок : 6 месяцев Количество, штук: 1 Код ТН ВЭД ЕАЭС: 8484900000 : 0.2",
        "customsCommodityCode": "8484900000",
        "pictures": [
          "https://avatars.mds.yandex.net/get-marketpic/13446293/pic6e699c13a051ae2778b539ddd14dfc35/orig"
        ],
        "weightDimensions": {
          "length": 0.1,
          "width": 0.1,
          "height": 0.01,
          "weight": 0.05
        }
      },
      "mapping": {
        "marketSku": 103757217881
      }
    }
  ]
}
```

Чтение: `POST …/v2/businesses/861370/offer-mappings` с `{ "offerIds": ["1107932"] }`.

---

## 2. Цена — пройден (весь кабинет)

| | |
|--|--|
| Method | **POST** |
| URL | `https://api.partner.market.yandex.ru/v2/businesses/861370/offer-prices/updates` |

Не `campaigns/137514772` — будет `LOCKED` / `Partner use only default price`.  
Поля `warehouseId` в цене нет: значение уходит **во все магазины** кабинета.

- [ ] HTTP **200**, `"status": "OK"`.
- [ ] В кабинете: цена **945**, зачёркнутая **1030** (лаг / карантин возможны).

```json
{
  "offers": [
    {
      "offerId": "1107932",
      "price": {
        "value": 945,
        "discountBase": 1030,
        "currencyId": "RUR"
      }
    }
  ]
}
```

`discountBase` должен быть на 5–75% выше `value` (1030 при 945 — ок).

---

## 3. Остаток — пройден (один склад)

| | |
|--|--|
| Method | **PUT** |
| URL | `https://api.partner.market.yandex.ru/v2/campaigns/137514772/offers/stocks` |

Ключ массива **`skus`** (не `skuItems`). Метод **PUT**.

Правила с прогона:

1. Один запрос = один `warehouseId`. Нужны все склады — повторить PUT (свой `campaignId` + свой склад из [таблицы](../evidence/external/warehouses-861370.md)).
2. **`updatedAt` = текущие часы Яндекс Маркета** (ISO 8601 со смещением, напр. `+05:00`). Если время в прошлом относительно уже лежащей записи — Маркет обновит **только дату**, **`count` не сменится**. Не копировать старый timestamp из примера.
3. Смотреть остаток у склада **1669219** / магазина 137514772, не на чужом ЕКБ и не на публичной витрине.

- [ ] HTTP **200**, `"status": "OK"`.
- [ ] В кабинете на складе «Ruli МСК 1 день»: **10** (или то `count`, что отправили с **свежим** `updatedAt`).
- [ ] Повтор с новым `count` и **новым** `updatedAt` «сейчас» — количество меняется.

Эталон тела (время в примере **не** копировать — подставить сейчас):

```json
{
  "skus": [
    {
      "sku": "1107932",
      "warehouseId": 1669219,
      "items": [
        {
          "type": "FIT",
          "count": 10,
          "updatedAt": "2026-09-02T13:07:00+05:00"
        }
      ]
    }
  ]
}
```

Проверка API: `POST …/v2/campaigns/137514772/offers/stocks`  
`{ "offerIds": ["1107932"], "stocksWarehouseId": 1669219 }`.

---

## 4. Сводка Postman

| Кейс | URL | Итог 2026-09-02 |
|------|-----|-----------------|
| Карточка | business `offer-mappings/update` | OK, кабинет обновился |
| Цена | business `offer-prices/updates` | OK, 945 / 1030 |
| Цена campaign | campaigns `offer-prices/updates` | **LOCKED** — не использовать |
| Остаток | campaign PUT `offers/stocks` + свежий `updatedAt` | OK, count меняется |
| Остаток, старый `updatedAt` | тот же PUT | OK, но count как был |
| Остаток `skuItems` на campaign | — | `skus size … 0` |

---

## 5. Что доделать, чтобы то же шло из Webasyst автоматически

Сейчас Postman ≠ плагин `ym`. CLI: `php {wa_root}/cli.php shop ym 17` (карточки/цены/остатки по интервалам профиля). UI шлёт **только** карточку.

### P0 — без этого автовыгрузка на 861370 ломается

| # | Разрыв | Сейчас в `ym` | Нужно как в Postman |
|---|--------|----------------|---------------------|
| 1 | URL цены | `POST campaigns/{campaign_id}/offer-prices/updates` | `POST businesses/{business_id}/offer-prices/updates` |
| 2 | Ключ цены | `"id"` | `"offerId"` (или оба, пока Маркет принимает) |
| 3 | `business_id` | пустой/неверный → `Incorrect businessId` | **861370** в «Идентификатор кабинета» профиля 17; campaign оставить **137514772** |
| 4 | Остаток `updatedAt` | `date('c')` сервера WA | Часы **сервера test.ruli.ru** = часовой пояс ЯМ; иначе count не едет. Проверить TZ/NTP |
| 5 | Один склад | одно поле `warehouse_id` на профиль | Для всех складов кабинета: маппинг `warehouseId` ↔ `campaignId` и цикл PUT (18 складов). Один профиль 17 закрывает только **1669219** |
| 6 | Узкий список CLI | `set_products` = пусто/`all` опасно | Постоянный сет SKU на ЯМ (не `all`); cron не по всему каталогу |
| 7 | Баг UI карточки | галка строки → нет hash → «без ошибок» **без** API | Либо чинить `getSelectedProducts` (hash `id/{product_id}`), либо карточку только CLI/очередью |

### P1 — тело карточки разъедется с эталоном

| # | Поле эталона | Плагин |
|---|----------------|--------|
| 8 | `offerId` | пишет `shopSku` |
| 9 | `mapping` сосед `offer` | кладёт `mapping.marketSku` **внутрь** `offer` |
| 10 | `customsCommodityCode` (строка) | `customsCommodityCodes` (массив) |
| 11 | `vendorCode` = `JB-22282-1107932` | `getVendorCodeByProductAndSku` — сверить, что формула та же |
| 12 | CLI карточки | ключ `offerMappingEntries`, не `offerMappings` |
| 13 | Фото | URL витрины Shop, не `avatars.mds.yandex.net` (Маркет может принять оба) |

### P2 — процесс / эксплуатация

| # | Что | Зачем |
|---|-----|--------|
| 14 | Интервалы CLI: карточки/цены (часы), остатки (минуты) > 0 + сброс last_time | Иначе cron молчит |
| 15 | Галка «Запись логов запросов» | `query.17.log` = доказательство, что ушло |
| 16 | `min_stock` / `min_price` | Ниже порога уйдёт **count: 0** |
| 17 | `discountBase` режется правилом 5–75% | Иначе зачёркнутой не будет |
| 18 | Цена только кабинета | Не ждать цену «только на 1669219» — API так не умеет |
| 19 | UI не шлёт цену и остаток | Автомат = **cron CLI** (или новая очередь), не кнопка «в Яндекс.Маркет» |
| 20 | Identification **0** | `offerId` = `shop_product_skus.id` = `1107932` |

### Минимальный контур автомата (после P0)

```text
cron → php cli.php shop ym 17
  → карточки: POST businesses/861370/offer-mappings/update   (offerId, узкий сет)
  → цены:     POST businesses/861370/offer-prices/updates    (не campaign)
  → остатки:  PUT  campaigns/{campaignId}/offers/stocks
              по каждому складу, updatedAt = now(TZ ЯМ)
```

Пока P0.1–P0.2 не сделаны, CLI цены на профиле 17 получит тот же **LOCKED**, что Postman на campaign.  
Пока P0.4–P0.5 не сделаны, CLI остатков либо не сменит count, либо обновит только склад из одного поля профиля.

---

## 6. Приёмка автомата (когда доделают код)

На том же SKU `1107932`, без Postman:

- [ ] Смена названия/описания в Shop → после CLI карточка в кабинете совпадает.
- [ ] Смена цены Shop → в кабинете **945-класс** на всех магазинах (не только МСК).
- [ ] Смена остатка Shop на складе, который мапится на **1669219** → count в кабинете на этом складе; остальные склады не обнулить случайно.
- [ ] В `query.17.log`: три URL как в §§1–3; в теле остатка свежий `updatedAt`.
- [ ] Повтор CLI без смены данных не затирает count из‑за старого timestamp.
