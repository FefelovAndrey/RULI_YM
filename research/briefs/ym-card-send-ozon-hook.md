# Карточка ЯМ: как собирается сейчас и можно ли встать в процесс Ozon

Статус: `draft`  
Дата: **2026-09-24**  
Код: WA `caroptics` · плагин `ym`, приложение `ozon`  
Связано: [cards-ozon-wb.md](../as-is/cards-ozon-wb.md), [ym-catalog-process-ba.md](ym-catalog-process-ba.md), [F-CATALOG](../../features/F-CATALOG/PRD.md), [D-YM-CARD-GEN](../../decisions/D-YM-CARD-GEN.md)

## Вывод

В процесс Ozon **вклиниться нельзя**. Боевой экспорт не публикует событие и на выходе отдаёт тело Seller API (`/v3/product/import`), а не карточку Shop.

Общее у двух контуров одно: **таблицы Shop** (`shop_product`, SKU, features, фото) и ключ оффера **`shop_product_skus.id`**. Сборщик, категория, единицы, hash, лимит и HTTP — свои. Это уже зафиксировано инвариантом I-1 в [ym-catalog-process-ba.md](ym-catalog-process-ba.md). Срез кода 2026-09-24 его подтверждает, не отменяет.

---

## Как ЯМ шлёт карточку сейчас

Два входа, одна сборка `shopYmApi::getDataForAddOfferByHash`, один POST `apiOfferMappingEntriesUpdates` → `POST /v2/businesses/{businessId}/offer-mappings/update`.

| Канал | Когда | Тело |
|-------|--------|------|
| CLI `php cli.php shop ym {profile}` | `cli_upload_products` > 0 и env = cli | ключ **`offerMappingEntries`** |
| UI «в Яндекс.Маркет» | long action `shopYmPluginBackendSendToYm` | ключ **`offerMappings`**; при ERROR повтор уже с `offerMappingEntries` |

Пустой `set_products` в CLI = коллекция `all`. Гейтов (фото, вес, `part_type`) и hash-skip на CLI нет. Категория — строка имени категории Shop или значение характеристики профиля (`getCategoryProductAndSku`), не `marketCategoryId`. `offer-cards` плагин не вызывает.

Поля сборки: `shopSku` (режим identification; 0 = `sku.id`), имя, vendor, vendorCode, barcodes, описание, ТН ВЭД массивом `customsCommodityCodes`, `mapping.marketSku` **внутри** `offer`, `weightDimensions`, urls, pictures.

Эталон — рабочий Postman 2026-09-02 / 2026-09-24: [offer-1107932-mappings.json.md](../evidence/external/offer-1107932-mappings.json.md). Чтение и запись — разные URL.

| | Чтение | Запись (прошла) |
|--|--------|-----------------|
| URL | `POST /v2/businesses/861370/offer-mappings` | `POST …/offer-mappings/update` |
| Тело | `{ "offerIds": ["1107932"] }` | `offerMappings[]` |

Запись, которую Маркет принял: `offer.offerId` = `1107932`, `vendorCode` = `JB-22282-1107932`, `barcodes` = `["JB-22282"]`, `customsCommodityCode` строкой, `mapping.marketSku` **рядом** с `offer`, фото — URL `avatars.mds.yandex.net`. Цены, `urls` и `marketCategoryId` в теле нет. Габариты `0.1 / 0.1 / 0.01 / 0.05` — заглушка, которую API принял; живые размеры Shop этот прогон не проверял.

Плагин это тело не собирает:

| Рабочий Postman | `getDataForAddOfferByHash` |
|-----------------|----------------------------|
| `offer.offerId` | `offer.shopSku` |
| ключ `offerMappings` | CLI: `offerMappingEntries`; UI при ошибке откатывается на него же |
| `mapping` сосед `offer` | `mapping` внутри `offer` |
| `customsCommodityCode` строка | `customsCommodityCodes` массив |
| `vendorCode` = артикул + `-` + sku_id | `getVendorCodeByProductAndSku`: `s.sku`, `s.name` или одна характеристика. Склейки `JB-22282-1107932` в коде нет |
| фото Market CDN | URL Shop; выборка картинок не индексируется по `product_id` |
| нет `urls`, нет строковой `category` | оба поля уходят, если заполнены |

Это контракт [F-CATALOG](../../features/F-CATALOG/PRD.md). Дерево категорий для этого обновления не нужно.

## Откуда берутся поля

Общий вход — карточка Shop (`shop_product`, SKU, характеристики, фото). Ozon после этого читает свои таблицы и шаблоны. Плагин `ym` читает только то, на что указывают настройки профиля. PIM и 1С в сборку карточки не входят.

Кого слать в сам атрибут не входит. Ozon берёт список `shop_set_products` `ozon-24`. Плагин `ym` — `set_products` профиля или весь каталог, если сет пуст.

| Поле прогона | Таблица и поле | Ozon | ЯМ, плагин сейчас |
|-----------------|---------------|------|-------------------|
| `offerId` — ваш id оффера в кабинете. В прогоне `1107932` | `shop_product_skus.id`. Справочника нет | Пишет это же значение в `offer_id` | Режим identification: 0 = `id`, 1 = `shop_product_skus.sku`, 2/3 = `shop_product.id`. В тело кладётся `shopSku`, не `offerId` |
| `name` — название на витрине | `shop_product.name`. Кэш Ozon: `ozon_products_name.value`. Шаблон: `ozon_defines` (`interface=generate_attributes`, `name`) | Сначала кэш и шаблон, иначе `shop_product.name` | Настройка `feature_name`: `shop_product.name` или `shop_product_features`. Можно дописать `shop_product_skus.sku` |
| `vendor` — бренд. В прогоне `STONE` | `shop_product_features` при `feature_id=52` (`proizvoditel`), текст в `shop_feature_values_*`. Справочник значений Ozon: `ozon_value_pair`, атрибут `ozon_attribute`, правило `ozon_product_rule` | Значение характеристики уходит атрибутом Ozon | Настройка `feature_vendor` → `shop_product_features`. Своего справочника брендов нет |
| `vendorCode` — артикул поставщика. В прогоне `JB-22282-1107932` | `shop_product_skus.sku` или `shop_product_skus.name`. Справочника артикулов нет | Отдельной таблицы нет. Артикул может уйти атрибутом, если есть строка в `ozon_product_rule` | Настройка `feature_vendorCode`: `sku`, `name` или одна характеристика. Склейки артикул–id SKU нет |
| `barcodes` — штрихкоды. В прогоне артикул `JB-22282` | `shop_product_features` по настройке. Справочника штрихкодов в сборщике нет | Атрибут Ozon, только если есть `ozon_product_rule` | Настройка `feature_barcode` → `shop_product_features` |
| `description` — текст карточки | `shop_product.description` / `summary`. Кэш: `ozon_products_description.value`. Шаблоны по виду запчасти: `ozon_parttype_description`, `ozon_parttype_description_template` | Кэш и шаблон по `part_type`, не сырое описание Shop | Настройка `feature_description`: `description`, `summary` или характеристика. Обрезка 3000 |
| `customsCommodityCode` — ТН ВЭД строкой. В прогоне `8484900000` | `shop_product_features` по настройке `feature_customsCommodityCodes`. Справочника ТН ВЭД нет | Атрибут Ozon, только если есть `ozon_product_rule` | Та же характеристика. Уходит массивом `customsCommodityCodes` |
| `pictures` — URL фото | `shop_product_images` (`product_id`, `filename`, `width`, `height`, `sort`). Дорисовка: `ozon_products_image_over`. Общие кадры: `ozon_products_image` где `type=general` | Фото Shop плюс обе таблицы ozon | `shop_product_images` (выборка не по `product_id`), инфографика канала `yandex`, URL из настроек профиля. Справочника нет |
| `weightDimensions.length` — длина | `shop_feature_values_dimension.value_base_unit`, связь `shop_product_features.feature_id=27`. Коэффициент: `ozon_defines` `ovh_size` | Миллиметры после × 1000 и `ovh_size` | Настройка `feature_length` → та же характеристика, без `ozon_defines` |
| `weightDimensions.width` — ширина | То же, `feature_id=29` | Миллиметры и `ovh_size` | Настройка `feature_width` |
| `weightDimensions.height` — высота | То же, `feature_id=28` | Миллиметры и `ovh_size` | Настройка `feature_height` |
| `weightDimensions.weight` — вес | `shop_feature_values_dimension.value_base_unit`, `feature_id=1`. Коэффициент `ozon_defines` `ovh_weight` | Граммы после × 1000 и `ovh_weight` | Настройка `feature_weight`, без коэффициента |
| `mapping.marketSku` — SKU карточки Маркета, рядом с `offer`. В прогоне `103757217881` | `shop_product_features` по настройке `feature_marketSku`. Справочника нет | Поля и таблицы нет | Настройка `feature_marketSku`. Кладётся внутрь `offer` |

В прогоне нет цены, `urls` и категории. Ozon цену кладёт в ту же карточку (индекс `ozon_exported_product_price_index` или расчёт от цены SKU), категорию берёт из `part_type` → `ozon_category_parttype`. Плагин `ym` цену в карточку не кладёт, а категорию пишет строкой имени категории Shop. Rich и применимость есть только у Ozon.

Рабочий Postman `1107932` по источникам не равен ни одному сборщику: имя и описание похожи на текст Shop, `vendorCode` склеен вручную, фото — уже залитый URL Маркета, габариты — заглушка.

Фото: выборка `shop_product_images` кладётся в плоский список, а код читает её как массив по `product_id` (`isset($products_images[$product_id])`). Инфографика `yandex` может дописать картинки отдельно. Надёжный источник фото для массовой выгрузки это не даёт.

Полный контент (категория ЯМ + `parameterValues`) — [D-YM-CARD-GEN](../../decisions/D-YM-CARD-GEN.md), статус **proposed**. В варианте A там ещё `offerId = Sklad`; живые заказы и Postman идут с `sku_id`. Принимать A в этом виде нельзя.

---

## Как Ozon собирает и шлёт карточку

Боевой cron: `ozonHashedExportProducts` (`wa-apps/ozon`).

```text
сет ozon-24 (карусель last product_id)
  → ozonProductsExport::getProductsByIds
       getOffers: SQL-гейт
       фото, характеристики, имя/описание, rich, цена, применимость
       generateOzonOffers → тело Seller API
  → отсев пустого type_id
  → hash ozon_products_hash (совпал — skip)
  → лимит POST /v4/product/info/limit (daily_update)
  → чанки 100 → POST /v3/product/import
  → task_id пишется в hash
```

Запасной контур без списка — `ozonExportProducts` (карусель всех валидных). Ручной экспорт из сета идёт через `ozonExportManager` в тот же `getProductsByIds`.

SQL-гейт `getOffers` (cron, без лога «битых» — лог только если есть пользователь):

- товар `status = 1`, цена SKU ≠ 0;
- обязательные JOIN: `part_type` (feature 4), вес (1), глубина/высота/ширина (27/28/29);
- категория: `ozon_category_parttype` / для truck — `ozon_category_truck_parttype`;
- `GROUP BY product_id` — одна строка на товар, `offer_id` = `shop_product_skus.id`.

Дальше CLI выкидывает оффер без `type_id`. В `generateOzonOffers` без основного фото и без правил атрибутов оффер не уходит.

Тело, которое уходит в Ozon, уже площадочное:

| Поле Ozon | Почему это не карточка ЯМ |
|-----------|----------------------------|
| `offer_id` | Совпадает по смыслу (`sku_id`), имя поля другое |
| `description_category_id` + атрибуты с id Ozon | Справочник `ozon_category*`, не `marketCategoryId` |
| `depth/height/width`, `weight` | Миллиметры и граммы, ещё коэффициент `ovh_size` / `ovh_weight` |
| `price`, `old_price`, `vat` | У ЯМ цена — отдельный `offer-prices/updates` на кабинет |
| `primary_image`, rich, видео 21841 | Пайплайн картинок приложения `ozon` |
| hash | Считает этот JSON, включая цену и атрибуты Ozon |

`wa()->event` на этом пути нет. Классы в `ozon/lib/handlers/` — Ozon **слушает** Shop (меню, заказ, карточка товара), наружу «оффер собран» не отдаёт. Kafka `mp_card_update` связывает WA↔1С в PIM и карточку МП не везёт.

---

## Куда теоретически вставать — и почему не встаём

| Точка | Вердикт |
|-------|---------|
| Подписка на событие после сборки Ozon | События нет |
| Вызов `getProductsByIds` и перекладка в `offer-mappings` | На выходе тело Ozon: чужая категория, мм/г, цена внутри карточки, id атрибутов Ozon. Зависимость от `ozon_*` и от правок сборщика |
| Строка отправки ЯМ внутри `ozonHashedExportProducts` после hash | Общий сет `ozon-24`, общий суточный лимит Ozon, общий чекпоинт `hashed_export_last_product_id`. Сбой или смена ассортимента одной площадки двигает другую |
| Тот же `shop_set` `ozon-24` как список ЯМ, без вызова кода Ozon | Ассортимент навсегда общий. Отдельный сет ЯМ уже в Q-009 |
| Те же таблицы Shop, свой сборщик, свой Partner API | Единственная рабочая граница |

Код Ozon не трогаем: ни CLI, ни `ozon_*`, ни копипаст `ozonProductsExport` с заменой имён. Паттерн процесса (список → гейты → hash → чанк → реестр → отдельно цена и остаток) уже расписан в БА, волны B0–B6.

---

## Варианты: тело Postman, источники как у Ozon

Статус: варианты, решение не принято. Дата: 2026-09-24.

Форма карточки — запись, которую Маркет уже принял (`offer-mappings/update`, `offerId` = `sku_id`, `mapping` рядом с `offer`, `customsCommodityCode` строкой, без цены и без `marketCategoryId`).

Чтение данных — те же сущности Shop, что в `ozonProductsExport::getOffers` / `addImages`: товар, SKU, `part_type`, вес, три габарита, бренд, `shop_product_images`. Не вызов `getProductsByIds` и не таблицы `ozon_*`.

Что у Ozon есть, а в рабочее тело ЯМ не переносим: id категории Ozon, атрибуты Ozon, миллиметры и граммы как есть, коэффициент `ovh_*`, цену внутри карточки, rich и применимость.

`vendorCode` Ozon не формирует. В прогоне это склейка `артикул-sku_id` (`JB-22282-1107932`), штрихкод — сам артикул (`JB-22282`). Это правило Postman, не правило Ozon.

| | V1. Свой сборщик | V2. Тексты из кэша Ozon | V3. Патч текущего `ym` | V4. Маппинг тела Ozon |
|--|------------------|-------------------------|------------------------|------------------------|
| Суть | Новый код ЯМ читает те же таблицы Shop и пишет тело Postman | Имя и описание из `ozon_products_name` / `ozon_products_description`, остальное как V1 | `getDataForAddOfferByHash` перевести на те же характеристики и на тело Postman | `getProductsByIds` → переложить в `offerMappings` |
| Плюс | Граница с Ozon чистая, форма API уже доказана | Текст на витрине совпадёт с Ozon | Быстрее всего доходит до одного SKU | Ноль своей сборки |
| Минус | Шаблоны имени и описание надо завести свои, либо брать `shop_product.name` / `description` | Карточка ЯМ ждёт, пока cron Ozon прогреет кэш; лимит имени Ozon 200 символов | Нет hash, реестра и запрета `all` — это не массовый контур | Единицы, цена и категория Ozon попадут в Маркет |

Рекомендация: **V1**. Первым срезом можно сделать **V3** на SKU `1107932`: то же тело, что в Postman, поля из тех же характеристик Shop. Массовый cron, свой сет и hash — уже слой B, не этот срез. **V2** и **V4** не брать.

Сборка V1 по полям прогона — та же таблица выше: смысл атрибута в столбце «Поле прогона», чтение как у Ozon, в тело ЯМ кладётся форма Postman, без коэффициента `ovh` и без цены. `vendorCode` и `barcodes` — по правилу прогона (артикул + id SKU и сам артикул), не по атрибуту Ozon.

Гейт как SQL Ozon, без его категории: товар включён, цена SKU не ноль, есть вид запчасти, вес, три размера, хотя бы одно фото. Нет связки `ozon_category_parttype` — для этой волны она не нужна.

Порядок для карточки ЯМ не меняется:

1. Починить тело и URL в текущем `ym` на узком сете — `F-CATALOG` (эталон Postman, без дерева категорий).
2. Отдельный сет, запрет `all`, hash, реестр — слой B.
3. `part_type` → `marketCategoryId` и `offer-cards` — только после переутверждения `D-YM-CARD-GEN` с `offerId = sku_id`.
