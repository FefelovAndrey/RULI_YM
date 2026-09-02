# Якоря: карточки / цены / остатки (плагин ym)

Код: `C:\Users\Дмитрий Ли\Projects\RULI\CodeBase\caroptics`  
Плагин: `wa-apps/shop/plugins/ym`  
Прогон: [catalog-price-stock-test.md](../../briefs/catalog-price-stock-test.md)

Секреты не копировать. OAuth в `shopYmApi::apiQuery` — из настроек профиля.

---

## Идентификатор

| Что | Где |
|-----|-----|
| shopSku / id цены / sku остатка | `shopYmApi::getShopSkuByProductAndSku` |
| Режимы 0–3 | `identification` в профиле |

Заказ 61089913793: `offerId`/`shopSku` = `1107932` (= `sku_id`, режим 0).

---

## Карточки

| Что | Где |
|-----|-----|
| Сборка оффера | `shopYmApi::getDataForAddOfferByHash` — имя, категория, фото, vendor, barcode, габариты |
| POST update | `shopYmApi::apiOfferMappingEntriesUpdates` → `/businesses/{businessId}/offer-mappings/update.json` |
| CLI | `shopYmCli`: только `wa()->getEnv() == 'cli'`; тело `offerMappingEntries` |
| UI | `shopYmPluginBackendSendToYm` — тело `offerMappings`; hash из POST |
| Список в кабинете | `apiOfferMappingEntries` → POST `/offer-mappings.json` |

---

## Цены

| Что | Где |
|-----|-----|
| Сборка | `getDataForAddPricesByHash` — `id` = shopSku, `price.value`, опц. `discountBase` |
| POST | `apiGetOfferPricesUpdates` → `/campaigns/{id}/offer-prices/updates.json` |
| Чтение | `apiGetOfferPrices` → GET `/offer-prices.json` |
| CLI | `cli_upload_prices`, чанк 50, кэш `upload_price/{shop_id}` |

---

## Остатки

| Что | Где |
|-----|-----|
| Сборка PUT | `convertPricesToStocks` — `sku`, `warehouseId`, `items[].type=FIT`, `count`; пороги `min_stock`/`min_price` |
| PUT | `setOffersStocks` → `/campaigns/{id}/offers/stocks.json` |
| CLI | `cli_upload_stocks`, чанк 2000 |
| Callback Маркета | `shopYmPluginFrontendApi` `action=/stocks` → `returnStock` |

---

## URL клиента

`getApiUrl`: campaign vs business; суффикс `.json`; auth OAuth (не Api-Key). То же, что на прогоне заказа.
