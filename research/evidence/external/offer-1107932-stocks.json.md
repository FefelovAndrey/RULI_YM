# Пример тела остатка — sku `1107932`

Прогон 2026-09-02: `GET /v2/businesses/861370/warehouses` → **`warehouseGroups: []`**.  
Официально это **вариант B** (v3 кабинета), не PUT campaign (тот — для групп складов). Плагин `ym` всё равно бьёт в campaign PUT.

Стенд профиля 17 / sandbox-заказ: campaign **137514772** = склад **`1669219`** («Ruli МСК 1 день», Реутов).  
Остальные 17 складов кабинета **не** трогать в этом тесте.

`count: 1` — как позиция в заказе. Для видимого теста можно `2`, потом вернуть. **Не слать 0**, если не скрываешь оффер.

Список складов (id ↔ campaign, без адресов): [warehouses-861370.md](warehouses-861370.md).

## B — слать это (нет групп складов)

| | |
|--|--|
| Method | **POST** |
| URL | `https://api.partner.market.yandex.ru/v3/businesses/861370/offers/stocks/update` |

```json
{
  "skuItems": [
    {
      "sku": "1107932",
      "partnerWarehouseId": 1669219,
      "count": 1,
      "updatedAt": "2026-09-02T12:00:00+05:00"
    }
  ]
}
```

Документация: [updateStocksOnPartnerWarehouses](https://yandex.ru/dev/market/partner-api/doc/ru/reference/stocks/updateStocksOnPartnerWarehouses).

## A — как плагин (если B отклонит / для сверки CLI)

| | |
|--|--|
| Method | **PUT** |
| URL | `https://api.partner.market.yandex.ru/v2/campaigns/137514772/offers/stocks` |

```json
{
  "skus": [
    {
      "sku": "1107932",
      "warehouseId": 1669219,
      "items": [
        {
          "type": "FIT",
          "count": 1,
          "updatedAt": "2026-09-02T12:00:00+05:00"
        }
      ]
    }
  ]
}
```

Auth тот же OAuth профиля 17. Токены не копировать в git.

## Сбой Postman 2026-09-02

`BAD_REQUEST` / `skus size must be between 1 and 2000 (rejected size: 0)` — это валидация **v2 campaign** `offers/stocks`: в теле нет массива **`skus`** (часто ушло тело B `skuItems` на URL A, или POST вместо PUT).

Не смешивать: `skuItems` только на v3 `…/stocks/update`. `skus` только на v2 campaign PUT.

## PUT campaign OK, кабинет не изменился (2026-09-02)

Тело A принято (`status: OK`). На сайте/карточке количество не сдвинулось.

На 861370 **нет групп складов**. Официально запись витрины — **B** (`POST v3 …/stocks/update`). PUT v2 для групп складов: OK не значит, что кабинет/витрина взяли это значение.

Дальше: прочитать `POST /v3/businesses/861370/offers/stocks`, затем слать **B** с новым `count` (не 1, если в кабинете уже 1). `updatedAt` лучше не слать или ставить «сейчас» — в примере было 12:00 при отправке ~13:04. Лаг UI до ~15 мин. Смотреть склад **1669219** / магазин campaign 137514772, не чужой ЕКБ и не публичный market.yandex.ru.
