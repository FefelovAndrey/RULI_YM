# Пример тела цены — offerId `1107932`

Прогон Postman 2026-09-02: `POST …/campaigns/137514772/offer-prices/updates` → **`LOCKED` / `Partner use only default price`**.  
У кабинета **861370** цена только **общая** (`onlyDefaultPrice`). Писать нужно в кабинет, не в магазин.

| | Неверно на этом стенде | Верно |
|--|--|--|
| Метод | POST | POST |
| URL | `/v2/campaigns/137514772/offer-prices/updates` | `/v2/businesses/861370/offer-prices/updates` |
| Док | [updatePrices](https://yandex.ru/dev/market/partner-api/doc/ru/reference/prices/updatePrices) | [updateBusinessPrices](https://yandex.ru/dev/market/partner-api/doc/ru/reference/prices/updateBusinessPrices) |

Плагин `ym` (`apiGetOfferPricesUpdates`) бьёт в **campaign**. На профиле 17 CLI, скорее всего, получит тот же `LOCKED`. YMWB уже ходил в business-URL.

`value` = **833** — как `itemsTotal` в заказе sandbox `61089913793`. Для видимого теста можно другое целое, потом вернуть 833.

Плагин кладёт идентификатор в **`id`**. В Postman — **`offerId`**.

```json
{
  "offers": [
    {
      "offerId": "1107932",
      "price": {
        "value": 833,
        "currencyId": "RUR"
      }
    }
  ]
}
```

Опционально зачёркнутая цена (`discountBase`): должна быть на 5–75% выше `value`, иначе плагин её выкидывает. Для 833 допустимый диапазон примерно 875–1457.

```json
{
  "offers": [
    {
      "offerId": "1107932",
      "price": {
        "value": 833,
        "discountBase": 999,
        "currencyId": "RUR"
      }
    }
  ]
}
```

Auth тот же OAuth профиля 17. Токены не копировать в git.
