# AS IS — карта систем (контур Яндекс Маркета в YMWB)

> **Вне скоупа RULI.** Канон: [../../context/01-systems.md](../../context/01-systems.md) + [fbs-fulfillment-wa.md](fbs-fulfillment-wa.md).

Статус: `draft`  
Источник: `repo: YMWB` · 2026-08-27

## Контур

```text
Оператор (браузер)
    → Flask UI :5050 (web_app.py)
        → SQLite marketplace_base.db / !YMWB.db / control_center.db
        → APScheduler run_marketplace_cycle
            → Google Sheets «КАЗНА» (СКЛАД, лист ЯМ)
            → Telegram (заказы / ошибки)
            → api.partner.market.yandex.ru  (3 метода)
Кабинет Яндекс Маркета  ← карточки, приёмка, отгрузка (человек, не API)
1С / PIM / MPS / WA     ← в YMWB не используются
```

## Touchpoints

| Touchpoint | Система | Поведение сейчас | Якорь |
|------------|---------|------------------|-------|
| Каталог офферов ЯМ | Flask `/table/yandex` + кабинет ЯМ | CRUD строк `Маркетплейс=yandex`; API карточек нет | `web_app.py:show_table` |
| Остаток витрины | Partner API | PUT stocks, sku=`Sklad` | `stock.py:ym_update` |
| Цена витрины | Partner API | POST offer-prices, offerId=`Sklad` | `price_updater_master.py:update_yandex` |
| Входящий заказ | Partner API poll | GET orders PROCESSING/STARTED | `order_notifications.py:get_orders_yandex_market` |
| Списание своего склада | Google Sheets СКЛАД | Минус «Наличие» по «Арт мой» | `order_notifications.py:update_stock` |
| Списание внешнего | SQLite | marketplace.Нал + prices.Наличие | `update_stock` |
| Операции сборки | Google Sheets ЯМ | A=FBS, M=На сборке | `write_order_to_gsheets` |
| Алерт оператору | Telegram | Новый заказ + изменение остатка | `notify_about_new_orders` |
| ON/OFF канала | stock_flags.json | Перед OFF — PUT нулей | `toggle_stock` + `send_zero_stocks_once` |
| Аудит цикла/заказов | control_center.db | processes, order_history, order_steps | `services/control_store.py` |

## Интеграции ЯМ

| Связь | Синхрон | Владелец | Методы |
|-------|---------|----------|--------|
| YMWB → YM stocks | sync HTTP PUT | `stock.ym_update` | `/campaigns/{campaign_id}/offers/stocks` |
| YMWB → YM prices | sync HTTP POST | `price_updater_master.update_yandex` | `/businesses/{businessId}/offer-prices/updates` |
| YM → YMWB orders | sync poll GET | `get_orders_yandex_market` | `/campaigns/{campaign_id}/orders` |
| YM → YMWB push | — | нет | webhooks не реализованы |

Auth: один `ym_token` (Bearer). Идентификаторы: `campaign_id` (остатки, заказы), `businessId` (цены). Склад ЯМ в payload не передаётся.

См. `../evidence/code/ymwb-yandex-market.md`.
