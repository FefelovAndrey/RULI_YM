# Эталон карточки — offerId `1107932`

Кабинет **861370**. Прогон Postman **2026-09-02**, тело подтверждено **2026-09-24**.  
Цена и остаток в этот JSON не входят.

Два разных запроса:

| | Чтение | Запись |
|--|--------|--------|
| Method | POST | POST |
| URL | `/v2/businesses/861370/offer-mappings` | `/v2/businesses/861370/offer-mappings/update` |
| Тело | `{ "offerIds": ["1107932"] }` | `offerMappings` ниже |

Запись, которая прошла (`status: OK`, кабинет обновился):

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
        "description": "Прокладка впускного коллектора MAZDA FAMILIA/323 ZL 99- производства STONE артикул JB-22282. … Код ТН ВЭД ЕАЭС: 8484900000 : 0.2",
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

Полный текст описания — в [postman-catalog-checklist.md](../../briefs/postman-catalog-checklist.md) §1. Пробелы в имени и в URL фото из вставки в чат 2026-09-24 — артефакт копирования; в чеклисте URL без крайних пробелов.

## Что прогон доказал и чего не доказал

| Факт | Следствие |
|------|-----------|
| `offerId` = `1107932` = `sku_id` | Не `Sklad`, не артикул `JB-22282` |
| `mapping` — сосед `offer` | Не поле внутри `offer` |
| `customsCommodityCode` — строка | Не массив `customsCommodityCodes` |
| `vendorCode` = `JB-22282-1107932`, штрихкод = `JB-22282` | Артикул и код поставщика — разные поля |
| `marketSku` = `103757217881` | Привязка к уже существующей карточке Маркета, не генерация |
| Нет `marketCategoryId`, нет цены, нет `urls` | Базовое обновление живёт без дерева категорий и без цены в теле |
| `weightDimensions` 0.1 / 0.1 / 0.01 / 0.05 | Маркет принял заглушку. Реальные габариты Shop этим телом не проверены |
| Фото — URL `avatars.mds.yandex.net` | Уже залитая картинка Маркета, не `wa-data` Shop |

Старая реконструкция с `shopSku`, `mapping` внутри `offer` и заглушкой `test.ruli.ru` — черновик до прогона. Канон — тело выше.
