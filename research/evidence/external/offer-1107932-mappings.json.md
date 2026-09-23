# Пример тела карточки — offerId / article `1107932`

Кабинет: `https://partner.market.yandex.ru/business/861370/assortment/offer-card?article=1107932` (без логина страница не отдаётся).

Собрано из заказа sandbox `61089913793` и выгрузки того же артикула (не живой GET offer-card). Цена и остаток **в этот JSON не входят**.

`POST https://api.partner.market.yandex.ru/v2/businesses/861370/offer-mappings/update.json`

UI плагина `ym` — ключ `offerMappings`. CLI — `offerMappingEntries`.

```json
{
  "offerMappings": [
    {
      "offer": {
        "shopSku": "1107932",
        "name": "Прокладка впускного коллектора MAZDA FAMILIA/323 ZL 99-   STONE JB-22282",
        "vendor": "STONE",
        "vendorCode": "JB-22282",
        "barcodes": ["JB-22282"],
        "description": "Прокладка впускного коллектора MAZDA FAMILIA/323 ZL 99- STONE JB-22282",
        "customsCommodityCodes": ["8484900000"],
        "mapping": {
          "marketSku": 103757217881
        },
        "urls": [
          "https://test.ruli.ru/"
        ],
        "pictures": [
          "https://test.ruli.ru/wa-data/public/shop/products/REPLACE.jpg"
        ],
        "weightDimensions": {
          "length": 0.1,
          "width": 0.1,
          "height": 0.01,
          "weight": 0.05
        }
      }
    }
  ]
}
```

| Поле | Откуда | Надёжность |
|------|--------|------------|
| `shopSku` | `offerId` / `shopSku` заказа, `article=` в URL кабинета | факт |
| `name` | `offerName` заказа | факт |
| `vendor` / `barcodes` / `vendorCode` | STONE, `JB-22282` из названия и заказа | факт (бренд из названия) |
| `customsCommodityCodes` | ТН ВЭД с той же артикульной карточки | с более ранней выгрузки |
| `mapping.marketSku` | «Артикул ЯМ» `103757217881` | с более ранней выгрузки; сверить в кабинете |
| `urls` / `pictures` / `weightDimensions` | **заглушки** | подставить с карточки Shop / кабинета |

Auth: OAuth профиля 17. Не слать, пока `business_id` = **861370**.
