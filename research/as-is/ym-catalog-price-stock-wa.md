# AS IS: карточки / цены / остатки ЯМ в Webasyst

Канон: **плагин `ym`**, не YMWB и не PIM.

Тестовый протокол: [catalog-price-stock-test.md](../briefs/catalog-price-stock-test.md).  
Якоря: [wa-ym-catalog-price-stock.md](../evidence/code/wa-ym-catalog-price-stock.md).

---

## Цепочка

```text
shop_product + sku (+ features, фото, цена, stock)
  → список set_products (или all)
  → getShopSkuByProductAndSku  → id оффера на ЯМ
  → карточка:  POST businesses/{businessId}/offer-mappings/update
  → цена:      POST businesses/{businessId}/offer-prices/updates
               (плагин ym бьёт в campaigns/… — на 861370 это LOCKED)
  → остаток:   PUT  campaigns/{campaignId}/offers/stocks
               (прогон 2026-09-02: работает при свежем updatedAt;
                один warehouseId на запрос; профиль ym — одно поле склада)
  ← опц. Маркет спрашивает склад: POST /ym_api/{profile}/?action=/stocks
```

Карточки Ozon/WB — другой контур (`ozon` / `wildb`). PIM не шлёт контент на ЯМ.

---

## Идентификатор оффера

Настройка профиля `identification`:

| Значение | Что уходит как sku / shopSku |
|----------|------------------------------|
| 0 | `shop_product_skus.id` |
| 1 | поле `sku` (артикул) |
| 2 / 3 | `product_id` или `product_id`+`s`+`sku_id` |

Заказ sandbox 2026-09-01: `offerId` = `shopSku` = `1107932` → на профиле 17, скорее всего, режим **0**. Это не колонка YMWB `Sklad` (Q-008).

---

## Кто запускает

| Канал | Карточки | Цены | Остатки |
|-------|----------|------|---------|
| CLI `shop ym {profile}` | да, если `cli_upload_products` > 0 и **только CLI** | да, `cli_upload_prices` | да, `cli_upload_stocks` |
| UI «Отправить в Яндекс.Маркет» | да (long action, hash товара/сета) | нет | нет |
| Callback Маркета `/stocks` | нет | нет | ответ остатка, не PUT |

Интервалы CLI: карточки/цены в **часах**, остатки в **минутах**. `0` = не грузить.

Остаток режется `min_stock` / `min_price`: ниже порога уходит **0**.

---

## Ограничения (до прогона)

- Массовый CLI на `all` опасен на живой кампании.
- Тело CLI карточек (`offerMappingEntries`) и UI (`offerMappings`) **разъехались** — проверить на одном SKU.
- Прогон заказа 01-09 в `query.17` **не** содержит stocks/prices/mappings.
- Postman 2026-09-02: цена только business; остаток campaign PUT ок, если `updatedAt` = часы ЯМ. Чеклист: [postman-catalog-checklist.md](../briefs/postman-catalog-checklist.md).
