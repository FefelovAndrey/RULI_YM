# Reality Brief

Статус: `draft`  
Канон: **Webasyst Shop + 1С + Partner API**. YMWB вне скоупа.

## RULI / WA (обновлено 2026-09-01)

Исполнение FBS **уже есть** в плагинах `ym` + `clabels` (+ `rulimpsupplies` на first-mile). Sandbox-прогон закрыл цепочку accept → сборка → `READY_TO_SHIP` → ярлык PDF → `SHIPPED`. First-mile на `fake`-заказе не гоняли.

Дыры, из-за которых оператор спотыкается (не «нет API»): нет строки `shop_ym_order_state` на accept → гейт ярлыка на Сборке закрыт; вкладка «Подбор» скрыта; first-mile фильтры по умолчанию пустые.

- Процесс: [../as-is/fbs-fulfillment-wa.md](../as-is/fbs-fulfillment-wa.md)
- Прогон + чеклист доработок: [fbs-test-2026-09-01.md](fbs-test-2026-09-01.md)
- Якоря: [../evidence/code/wa-ym-fbs-pack.md](../evidence/code/wa-ym-fbs-pack.md)

## YMWB (историческое, не AS IS RULI)

Снято `2026-08-27` с [torsaf/YMWB](https://github.com/torsaf/YMWB). Не использовать как контур поставки.

## Как сейчас устроено (YMWB)

YMWB — операционный контур продавца: локальный каталог в SQLite, приоритет склада Sklad над внешними поставщиками, публикация остатков и цен на Яндекс Маркет, опрос новых заказов раз в ~5 минут, уведомление в Telegram и строка «На сборке» в Google Sheets.

Яндекс Маркет подключён **тремя** методами Partner API. Каталог карточек и весь фулфилмент после «заказ появился» живут в кабинете и таблицах, не в API-слое.

Ключ стыковки с ЯМ: `Sklad` = `sku` / `offerId`. Auth: Bearer `ym_token`. Скоуп остатков/заказов — `campaign_id`; скоуп цен — `businessId`.

## Где болит (YMWB, не RULI)

- Обрыв процесса на «На сборке»: нет accept/ship/label/cancel/return.
- Каталог не синхронизируется с кабинетом — ломается молча при опечатке артикула.
- Polling + узкий фильтр статусов.
- Списание остатка без фильтра маркетплейса (риск задеть WB/Ozon).
- Нет склада в payload ЯМ; FBS не подтверждается API.
- HTTP-клиент `clients.http_client` не лежит в git.

## Что технически возможно без героизма (YMWB, не план RULI)

- Оставить YMWB как источник витрины (stocks/prices) и детектор заказа.
- Нарастить контур заказа отдельными Partner API (orders accept, labels, status) — в коде точек расширения нет, только новый модуль.
- Подключить RULI/1С как SoT склада вместо Google Sheets — ломает `update_sklad` / `update_stock`.

## Evidence (ссылки)

- RULI FBS: `../as-is/fbs-fulfillment-wa.md`, `fbs-test-2026-09-01.md`, `../evidence/code/wa-ym-fbs-pack.md`
- YMWB (архив): `../as-is/process.md`, `../as-is/systems-map.md`, `../evidence/code/ymwb-yandex-market.md`
- data: `../as-is/data-readiness.md`

## Гипотезы на Options

1. **D-PUSH-VS-POLL** — callback `/order/accept` vs Postman/poll (на тесте accept вручную).
2. **D-STOCK-SOURCE** — Google Sheets vs 1С/RULI как SoT остатка для ЯМ.
3. **D-YM-FULFILLMENT** — **частично снято:** приёмка / ярлык / SHIPPED в WA уже есть; открыты first-mile, запись `ym_order_state`, живой FBS без sandbox.
