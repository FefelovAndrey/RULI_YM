# Evidence: генерация карточек Ozon / Wildberries в WA

Источник: `repo: WA / caroptics` (`C:\Users\Дмитрий Ли\Projects\RULI\CodeBase\caroptics`)  
Дата: `2026-08-27`

PIM **не** создаёт карточки МП. Топик `ru.ruli.products.wa.mp_card_update` — только маппинг `id_1c ↔ wa_id` в PIM после создания товара в WA.

## Ozon

```text
repo: WA / caroptics
path: wa-apps/ozon/lib/cli/ozonHashedExportProducts.cli.php
symbol: ozonHashedExportProductsCli
note: Боевой контур обновления. Список shop_set_products set_id='ozon-24'.
      Лимит POST /v4/product/info/limit (daily_update). Чанки по 100.
      Hash-skip: ozon_products_hash. Отправка ozonApi.exportProducts → POST /v3/product/import
```

```text
repo: WA / caroptics
path: wa-apps/ozon/lib/cli/ozonExportProducts.cli.php
symbol: ozonExportProductsCli
note: Карусель обновления без списка: товары status=1, есть фото,
      заполнены вес/габариты (feature 1,27,28,29) и part_type (4),
      есть связка ozon_category_parttype. Offset в ozon_defines.
```

```text
repo: WA / caroptics
path: wa-apps/ozon/lib/classes/product_handler/ozonProductsExport.class.php
symbol: getProductsByIds
note: offer_id = shop_product_skus.id. Категория Ozon через part_type → ozon_type_id /
      description_category_id. Обязательные: part_type, weight, size, brand.
      Rich-контент, применимость, общие картинки/видео, kфц ovh_weight/ovh_size.
```

```text
repo: WA / caroptics
path: wa-apps/ozon/lib/classes/api/ozonProductService.class.php
symbol: import
note: POST https://api-seller.ozon.ru/v3/product/import
      Статус задачи: POST /v1/product/import/info
```

## Wildberries

```text
repo: WA / caroptics
path: wa-apps/shop/plugins/wildb/lib/cli/shopWildbNewCardsExport.cli.php
symbol: shopWildbNewCardsExportCli
note: Создание новых карточек. Списки plugin setting shop_products_sets.
      Кандидат: в сете, status=1, sku.available=1, price>0, есть part_type
      со связкой shop_wildb_part_sync, ещё нет в shop_wildb_products_exist.
      Лимит GET content/v2/cards/limits. Чанк 100.
      POST content-api.wildberries.ru/content/v2/cards/upload
      Затем картинки + Kafka sku_ids mp=wb на цены/остатки.
```

```text
repo: WA / caroptics
path: wa-apps/shop/plugins/wildb/lib/classes/product_handler/shopWildbProductsExport.class.php
symbol: getOffersByIds / generateExportOffers
note: vendorCode = offer_id (sku_id). subjectID из маппинга категории.
      Характеристики shop_feature → wb_feature_id. Бренд через shop_wildb_brand_comparisons.
```

```text
repo: WA / caroptics
path: wa-apps/shop/plugins/wildb/lib/classes/api/shopWildbApiModule.class.php
symbol: cardsUpload / cardsUpdate / cardsUploadAdd
note: content/v2/cards/upload | update | upload/add | media/file | get/cards/list | error/list
```

## PIM (не генерация карточки МП)

```text
repo: PIM / fastapi-pim
path: app/infra/kafka/consumers/wa_product_mapping_update/consumer.py
symbol: consumer
note: Kafka ru.ruli.products.wa.mp_card_update → Create/Delete WAProductMapping (id_1c, wa_id)
```
