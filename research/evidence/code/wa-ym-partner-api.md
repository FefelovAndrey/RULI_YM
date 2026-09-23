# Partner API, как его вызывает плагин `ym`

База: `https://api.partner.market.yandex.ru/v2/`  
Класс: `shopYmApi` (`wa-apps/shop/plugins/ym/lib/classes/shopYmApi.class.php`).  
Auth: заголовок  
`Authorization: OAuth oauth_token="{oauth_token}", oauth_client_id="{client_id}"`  
(не Api-Key). Если oauth/client_id пустые — запрос **не** уходит, лога нет.

Сбор URL: `getApiUrl($path, $campaigns = true, $business = false)`:

| Режим | Шаблон |
|-------|--------|
| кампания (по умолчанию) | `…/v2/campaigns/{campaign_id}{path}.json` |
| кабинет | `…/v2/businesses/{business_id}{path}.json` |
| корень | `…/v2{path}.json` |

`campaign_id` ≠ `business_id`. Стенд профиля 17: campaign **137514772**, кабинет **861370**. Карточки идут на **business**. Заказы/цены/остатки — на **campaign**. Ошибка `Incorrect businessId: offer-mappings` = в «Идентификатор кабинета» не 861370.

Суффикс `.json` — особенность плагина, не актуальная рекомендация Partner API v2.

Лог `query.{profile}.log` только при галке «Запись логов запросов». Ошибки — `error.{profile}.log` всегда.

---

## 1. Карточка (отправка оффера)

То, что делает UI «в Яндекс.Маркет» и CLI `cli_upload_products`.

| | |
|--|--|
| Метод | `POST` |
| URL | `/v2/businesses/{business_id}/offer-mappings/update.json` |
| PHP | `apiOfferMappingEntriesUpdates` |

Тело UI (`sendToYm`):

```json
{
  "offerMappings": [
    { "offer": { "shopSku": "…", "name": "…", "…" : "…" } }
  ]
}
```

Тело CLI (`shopYmCli`): ключ **`offerMappingEntries`**, та же структура оффера.

Поля `offer` собирает `getDataForAddOfferByHash`: `shopSku`, `name`, `category`, `manufacturer`, `manufacturerCountries`, `weightDimensions` (length/width/height/weight), `urls`, `pictures`, `vendor`, `vendorCode`, `barcodes`, `description`, `shelfLife`, `lifeTime`, `guaranteePeriod`, `customsCommodityCodes`, опц. `mapping.marketSku`.

`shopSku` = `getShopSkuByProductAndSku` (identification 0 → `sku_id`).

Чтение списка офферов в кабинете (не создание):

| | |
|--|--|
| Метод | `POST` |
| URL | `/v2/businesses/{business_id}/offer-mappings.json` |
| PHP | `apiOfferMappingEntries` |

На стенде 2026-09-01: `BAD_REQUEST` / `Incorrect businessId: offer-mappings` — URL собрался, id кабинета неверный.

**Один URL не закрывает цену и остаток.** Карточка → business `offer-mappings/update`. Цена на 861370 → business `offer-prices/updates` (campaign LOCKED). Остаток → campaign `offers/stocks` **или** business v3 `offers/stocks/update`, если нет групп складов. Связка трёх тел — один и тот же `shopSku` / `offerId` / `sku`.

---

## 2. Цены

CLI `cli_upload_prices`. Из UI **не** шлётся.

| | |
|--|--|
| Запись (плагин) | `POST /v2/campaigns/{campaign_id}/offer-prices/updates.json` |
| PHP | `apiGetOfferPricesUpdates` |
| Тело плагина | `{ "offers": [ { "id": "<shopSku>", "price": { "currencyId": "RUR", "value": …, "discountBase": … } } ] }` |
| Чанк | 50 |
| Чтение | `GET /v2/campaigns/{campaign_id}/offer-prices.json` → `apiGetOfferPrices` |

`id` / `offerId` цены = тот же shopSku, что у карточки.

**Стенд 861370 (2026-09-02):** campaign-URL отвечает `LOCKED` / `Partner use only default price`. Рабочий:  
`POST /v2/businesses/861370/offer-prices/updates` с `"offerId"`, эталон `value: 945`, `discountBase: 1030`. Плагин на campaign — CLI цены без смены URL получит LOCKED. Цена не привязана к `warehouseId`.

---

## 3. Остатки

CLI `cli_upload_stocks`. Из UI **не** шлётся.

| | |
|--|--|
| Запись (плагин) | `PUT /v2/campaigns/{campaign_id}/offers/stocks.json` |
| PHP | `setOffersStocks` |
| Тело плагина | `{ "skus": [ { "sku": "<shopSku>", "warehouseId": <int>, "items": [ { "type": "FIT", "count": <int>, "updatedAt": "<ISO8601>" } ] } ] }` |
| Чанк | 2000 |

Ниже `min_price` / `min_stock` в профиле `count` становится 0.

Официально на 861370 (2026-09-02): **`warehouseGroups: []`** →  
`POST /v3/businesses/861370/offers/stocks/update`  
тело `{ "skuItems": [ { "sku": "1107932", "partnerWarehouseId": 1669219, "count": 1, "updatedAt": "…" } ] }`.  
Склад тестовой кампании 137514772 = **1669219** (Ruli МСК 1 день). Список: [warehouses-861370.md](../external/warehouses-861370.md).

Плагин `ym` шлёт `PUT /v2/campaigns/{campaign_id}/offers/stocks.json` — на прогоне 2026-09-02 этот метод **принял** count, если `updatedAt` совпал с часами ЯМ. Старый timestamp → OK, но count не меняется. Один `warehouseId` на запрос; для всех складов — цикл. Чеклист: [postman-catalog-checklist.md](../../briefs/postman-catalog-checklist.md).

Обратный вызов Маркета (не PUT): `POST /ym_api/{profile}/?action=/stocks` → `returnStock`.

---

## 4. Заказы (плагин → Маркет)

Кампания `campaign_id`. Прогон 01-09: campaign `137514772`.

| Назначение | HTTP | Path | PHP |
|------------|------|------|-----|
| Карточка заказа | GET | `/orders/{id}.json` | `getOrder` |
| Список | GET | `/orders.json` | `apiGetOrders` |
| Статус (READY_TO_SHIP, SHIPPED, …) | PUT | `/orders/{id}/status.json` | `changeOrderStatus` |
| Ярлыки PDF | GET | `/orders/{id}/delivery/labels.json` | labels |
| Короба | PUT | `/orders/{id}/boxes.json` | boxes |
| Короба поставки | PUT | `/orders/{id}/delivery/shipments/{sid}/boxes.json` | |
| Тело статуса | | `{ "order": { "status": "PROCESSING", "substatus": "READY_TO_SHIP" } }` | |

Маркет → WA (не Partner URL):  
`POST /ym_api/{profile}/?action=/order/accept?auth-token=`  
`POST …/order/status`  
`POST …/cart`

---

## 5. First-mile (тот же `shopYmApi` + `rulimpsupplies`)

Кампания.

| HTTP | Path |
|------|------|
| PUT | `/first-mile/shipments.json` |
| GET | `/first-mile/shipments/{id}.json` |
| GET | `/first-mile/shipments/{id}/act.json` |
| POST | `/first-mile/shipments/{id}/confirm.json` |
| GET | `/first-mile/shipments/{id}/orders/info.json` |
| POST | `/first-mile/shipments/{id}/excludeOrders.json` |
| GET | `/shipments/reception-transfer-act.json` |

`rulimpsuppliesYmApi` бьёт в ту же базу **без** `/v2` и **без** `.json` на части путей (`/campaigns/{id}/first-mile/shipments`, …).

---

## 6. Прочее исходящее `shopYmApi`

| HTTP | Path | Зачем |
|------|------|--------|
| GET | `/returns.json` | возвраты |
| GET | `/delivery/services.json` | службы доставки (корень, не campaign) |
| PUT | `/orders/{id}/verifyEac.json` | код ЕАС |
| PUT | `/orders/{id}/identifiers.json` | идентификаторы |
| GET | `/orders/{id}/buyer.json` | покупатель |
| POST | `/orders/{id}/delivery/track.json` | трек |
| POST | `/orders/{id}/deliverDigitalGoods.json` | цифра |
| GET | `/regions.json`, `/regions/{id}/children.json` | регионы |
| GET | `/outlets.json`, `/outlets/{id}.json` | ПВЗ |
| GET | `/campaigns.json` (busts, особый вызов) | кабинеты |

OAuth-токен: `POST https://oauth.yandex.ru/token` (не Partner).

---

## Кто что запускает

| Данные | Куда | Как запустить |
|--------|------|----------------|
| Карточка | business `offer-mappings/update` | список `#/id/{product_id}/` + «выбрать все» + «в Яндекс.Маркет»; или CLI `php cli.php shop ym {профиль}` |
| Цена | campaign `offer-prices/updates` | только CLI |
| Остаток | campaign `offers/stocks` | только CLI |
| Статус заказа | campaign `orders/{id}/status` | Сборка / карточка заказа YM |

CLI: `php {wa_root}/cli.php shop ym {profile_id}` (на стенде 17).
