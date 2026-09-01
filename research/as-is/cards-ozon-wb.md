# AS IS — генерация и отправка карточек Ozon и Wildberries

Статус: `draft`  
Источник: WA `caroptics` · evidence: `../evidence/code/wa-ozon-wb-cards.md`  
Обновлено: `2026-08-27`

## Суть

Карточка МП **собирается в WebAsyst** из карточки Shop-Script и уходит **прямым Seller/Content API**.  
PIM, MPS, YMWB, 1С в этом шаге **не вызывают** API карточек.

| Кто | Роль |
|-----|------|
| WA `shop_product` + SKU + features | Источник названия, фото, веса, габаритов, бренда, вида запчасти |
| Списки (`shop_set_products`) | Кого выгружать (Ozon: `ozon-24`; WB: настройки плагина) |
| Таблицы маппинга | part_type → категория МП; feature WA → attribute/charc МП |
| Приложение `ozon` / плагин `wildb` | Сборка payload, лимиты, чанки, ошибки |
| Kafka `mp_card_update` | Только связать товар WA с PIM (`id_1c` ↔ `wa_id`), не контент МП |
| MPS / odata1c | Цены и остатки **после** того, как карточка уже есть на МП |

Идентификатор оффера на обеих площадках: **`shop_product_skus.id`**.  
Это **не** колонка `Sklad` из YMWB — для ЯМ это открытый разрыв идентичности.

---

## Happy path Ozon

```text
Список ozon-24 (hashed)  ИЛИ  карусель всех валидных товаров
  → отбор: status=1, фото, вес, ГxШxВ, part_type, связка категории Ozon
  → ozonProductsExport: offer_id = sku_id, description_category_id, type_id,
        attributes, images, rich-контент, применимость
  → hash: если payload не менялся — skip
  → лимит daily_update (POST /v4/product/info/limit)
  → чанки 100 → POST /v3/product/import
  → task_id → (отдельно) /v1/product/import/info
```

Обязательные характеристики WA: `part_type`, `weight`, `size`, `proizvoditel`.

## Happy path Wildberries

```text
Списки shop_products_sets
  → кандидат на CREATE: нет строки в shop_wildb_products_exist
  → лимит content/v2/cards/limits
  → shopWildbProductsExport: subjectID, vendorCode=sku_id, title/description,
        characteristics, dimensions, brand
  → чанки 100 → POST content/v2/cards/upload
  → запись exist + картинки content/v2/media/file
  → Kafka: {sku_ids, mp:"wb"} на пересчёт цен и остатков
```

Обновление уже существующих карточек WB — `content/v2/cards/update` / UI `CardsExport` (старый long-action) плюс CLI создания.

## Ветвления

| Ветка | Ozon | WB |
|-------|------|-----|
| Нет лимита | цикл выходит | цикл выходит |
| Нет категории/type | оффер отсеивается | нет part_sync — не в выборке |
| Hash совпал | skip | нет hash на create (есть exist-таблица) |
| vendorCode уже на WB | — | подтянуть exist, не дублировать |
| Картинка неквадратная / &lt;200 | ресайз/отсев | отдельные media после upload |

## GAP к ЯМ (факт)

YMWB шлёт остатки/цены с `offerId = Sklad`. Карточки Ozon/WB живут с `offer_id = sku_id`. Без решения идентичности контур ЯМ разъедется.
