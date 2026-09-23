# D-YM-CARD-GEN — генерация карточек Яндекс Маркета по образцу Ozon/WB

Статус: `proposed`  
Дата: 2026-08-27

## Контекст

Карточки Ozon и WB сегодня генерирует **WA** и отправляет Seller/Content API. PIM только хранит маппинг WA↔1С. YMWB карточек ЯМ **не** создаёт.

Нужен тот же класс процесса для ЯМ: отбор из каталога WA → маппинг категории/атрибутов → Partner API → затем уже цены/остатки (YMWB или MPS).

Связанный разрыв: YMWB использует `offerId = Sklad`, Ozon/WB — `sku_id`. См. Q-008.

## Варианты

### A — Плагин/приложение WA (рекомендуемый)

Повторить контур `wa-apps/ozon` + `wildb`:

1. Список товаров (`shop_set_products`, напр. `yandex-1`).
2. Таблицы маппинга: `part_type` → `marketCategoryId`; feature WA → `parameterId` ЯМ.
3. Сборка payload из той же карточки Shop-Script (имя, описание, фото, вес, габариты, бренд, сертификаты).
4. Создание/обновление оффера: `POST /v2/businesses/{businessId}/offer-mappings/update`.
5. Категорийные характеристики: `POST /v2/businesses/{businessId}/offer-cards/update`.
6. Справочники: `POST /v2/categories/tree`, `POST /v2/category/{categoryId}/parameters`.
7. Статус карточки: `POST /v2/businesses/{businessId}/offer-cards`.
8. Hash-skip как Ozon hashed export; чанки с учётом лимита ЯМ (content ~5000/мин, mappings update — по доке кабинета).
9. После успешного create — Kafka `{sku_ids, mp:"ym"}` на цены/остатки (как WB).

`offerId` (решение внутри A): **на старте = `Sklad`**, чтобы не ломать текущий YMWB и уже заведённые в кабинете офферы. Параллельно вести таблицу `sku_id ↔ Sklad ↔ ym_offerId`. Когда RULI заменит YMWB — можно сойтись на `sku_id`.

Плюсы: тот же операционный паттерн, те же люди/списки/обязательные поля, не тащим генерацию в PIM/YMWB.  
Минусы: ещё один WA-контур (как ozon-app); маппинг категорий ЯМ нужно вести с нуля.

### B — Генерация в PIM

PIM собирает канон и шлёт Partner API сам.

Плюсы: один контент-хаб.  
Минусы: сейчас PIM этого не делает даже для Ozon/WB; сорвёт AS IS; нужен полный перенос rich/применимости/фото из WA.

### C — Ручной кабинет + YMWB как сейчас

Плюсы: ноль разработки.  
Минусы: не масштабируется; GAP каталога остаётся.

## Решение

Выбран: **A (proposed, не accepted)**  
Почему: максимальная аналогия с работающим контуром Ozon/WB при минимальном взрыве идентичности с YMWB.

## Implications

- Затрагивает фичи: `F-CATALOG` (W1), затем `F-PRICE-SYNC` / `F-STOCK-SYNC`
- Затрагивает правила: `R-OFFER-ID` (нужно завести, когда D принят)
- Откат: выключить список ЯМ; карточки в кабинете остаются

## Связи

- AS IS карточек: `../research/as-is/cards-ozon-wb.md`
- Reality Brief: `../research/briefs/reality-brief.md`
- Узел графа: `D-YM-CARD-GEN`
