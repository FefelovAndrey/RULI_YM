# ЯМ vs Ozon: карточки / цены / остатки на весь каталог

Дата: **2026-09-03**. Канон RULI: плагин `ym` + Partner API. Образец процесса: приложение `ozon` (`wa-apps/ozon`), не YMWB.

Контракты патча `ym` (один SKU, Postman): [F-CATALOG](../../features/F-CATALOG/PRD.md), [F-PRICE-SYNC](../../features/F-PRICE-SYNC/PRD.md), [F-STOCK-SYNC](../../features/F-STOCK-SYNC/PRD.md).  
Эталон API: [postman-catalog-checklist.md](postman-catalog-checklist.md).  
AS IS Ozon: [cards-ozon-wb.md](../as-is/cards-ozon-wb.md).

**Не копировать Ozon один в один:** цена ЯМ — на весь кабинет, не на склад; `offerId` на прогоне = `sku_id`, не `Sklad`.

---

## Как у Ozon (боевой паттерн)

| Поток | Кто | Кого берёт | Как шлёт |
|-------|-----|------------|----------|
| Карточка | `ozonHashedExportProducts` | список **`ozon-24`**, не весь Shop | гейты (фото, вес, габариты, `part_type`, категория Ozon) → **hash-skip** → лимит суток → чанки 100 → `/v3/product/import` |
| Карточка (карусель) | `ozonExportProducts` | все валидные без списка | offset в `ozon_defines`; те же гейты |
| Цена | `ozonUpdatePricesSimple` | уже выгруженные офферы (+ опц. один `sku_id`) | отдельный CLI; батчи; сверка с индексом цен Ozon |
| Остаток | `ozonUpdateStocksSimple*` | **склад Ozon** × списки Shop × склады Shop | цикл складов; дельта new/change/delete; 0 только точечно; реестр `ozon_exported_products_sku` |

Ключ оффера: **`shop_product_skus.id`**. PIM карточки МП не шлёт.

---

## Как у `ym` сейчас

Один CLI: `php cli.php shop ym {profile}` — карточки + цены + остатки по таймерам профиля.

| Поток | Сейчас | Чем хуже Ozon |
|-------|--------|----------------|
| Кого | `set_products` или **`all`** | пустой сет = весь каталог, без гейтов и hash |
| Карточка | UI (часто фейковый успех) / CLI ключ `offerMappingEntries` | нет `ozon-24`, нет hash, нет `part_type`→категория ЯМ, нет `offer-cards` |
| Цена | CLI на **campaign** + поле `"id"` | на 861370 = **LOCKED**; UI цены нет |
| Остаток | один `warehouse_id` на профиль, `updatedAt` = часы WA | нет таблицы складов как `ozonWarehouse`; 18 складов кабинета не закрыть одним профилем 17 |
| Реестр | нет аналога `ozon_exported_products_*` | нельзя «только то, что уже на МП» и точечный 0 |
| После create | ничего | у WB после карточки Kafka на цены/остатки; у ЯМ нет |

Partner API при этом **умеет** то, что Postman доказал на одном SKU.

---

## Что доработать (чтобы «все товары» = как Ozon)

Два слоя. Слой A — иначе CLI на кабинете **не работает даже на одном SKU**. Слой B — масштаб каталога.

### A — контракт API (уже в PRD, без этого cron бессмыслен)

- [ ] Цена: URL **business** `861370`, ключ **`offerId`** (`F-PRICE-SYNC`).
- [ ] Карточка: тело как Postman (`offerMappings`, `offerId`, `mapping` снаружи, `customsCommodityCode`); починить hash UI (`F-CATALOG`).
- [ ] Остаток: свежий `updatedAt` в TZ ЯМ; PUT `skus` (`F-STOCK-SYNC`).
- [ ] `business_id` профиля = 861370.

### B — процесс как Ozon (этого в `ym` нет)

Бизнес-анализ контура **с нуля**, без кода Ozon: [ym-catalog-process-ba.md](ym-catalog-process-ba.md).

| # | Как у Ozon | Что сделать в ЯМ |
|---|------------|------------------|
| 1 | Список `ozon-24` | Постоянный `shop_set` (Q-009, условно `yandex-1`). **Запрет `all` на бою.** |
| 2 | Гейты: фото, вес, Г×Ш×В, `part_type`, категория | Не слать «дырявые» SKU; ошибки в лог/сет `_check`, как `set_products_check` |
| 3 | `ozon_products_hash` | Hash payload карточки; без изменений — skip (лимиты ЯМ content) |
| 4 | `part_type` → категория Ozon | `D-YM-CARD-GEN`: `part_type` → `marketCategoryId` + `offer-cards/update` / `parameterValues`. Без этого массовая карточка = имя/фото, не полноценный контент. D ещё **proposed** |
| 5 | Реестр выгруженных | Таблица `sku_id` ↔ `offerId` ↔ статус карточки; цены/остатки **только** по ней |
| 6 | Отдельные CLI | Карточки / цены / остатки разными cron (у Ozon так). Сейчас один `shop ym` |
| 7 | Склад Ozon: списки + `stock_id` Shop | Справочник: `warehouseId` ЯМ + `campaignId` + склады Shop + сет. Цикл PUT. Не 18 складов «сразу наобум» (Q-007) |
| 8 | Дельта остатка | Сверять с последним отправленным count; 0 — только исчезновение, не обнуление каталога |
| 9 | Чанки + карусель | Карточки маленькими батчами + last_id; цены чанк 50 уже есть; остатки 2000 — ок, но по складу |
| 10 | После новой карточки | Триггер цены и остатка по `sku_id` (как Kafka `mp:wb`), не ждать общий cron |

### Не копировать с Ozon

- Цена на склад — у ЯМ **нет**; одна цена кабинета.
- `offerId` = `Sklad` из старого `D-YM-CARD-GEN` A — **конфликт** с заказами `1107932` / `726138`. Оставить identification **0**.
- Карусель «весь Shop без списка» — у Ozon есть запасной CLI; на ЯМ сначала только сет.

---

## Порядок работ

```text
1. Патч A (цена business, тело карточки, updatedAt)
2. Сет ЯМ + запрет all          ← иначе нельзя включать cron
3. Cron карточки по сету (hash)
4. Cron цены по реестру/сету
5. Маппинг 1–N складов (сначала 1669219)
6. Категории ЯМ (D-YM-CARD-GEN), когда контент перестанет хватать
```

Пока нет (1)+(2), «выгрузить все товары» = риск LOCKED, нулевых остатков и фейкового UI.
