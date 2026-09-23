# Evidence: YMWB × Яндекс Маркет

Источник: `repo: YMWB` (`https://github.com/torsaf/YMWB`), разбор 2026-08-27.  
Область: **только Яндекс Маркет**. WB/Ozon — только как общий каркас цикла.

## Что это

Однопроцессный Python-монолит (Flask + APScheduler + SQLite). Для ЯМ реализует **тонкий outbound-sync + inbound poll заказов**. Каталог карточек, приёмка, отгрузка, ярлыки, статусы заказа в кабинете, возвраты — **вне кода**.

## Якоря API (полный список вызовов к Partner API)

```text
repo: YMWB
path: stock.py
symbol: ym_update
note: PUT https://api.partner.market.yandex.ru/campaigns/{campaign_id}/offers/stocks
      Auth: Authorization Bearer {ym_token}
      Body: {"skus":[{"sku":"<Sklad>","items":[{"count":<Нал>,"updatedAt":"<ISO8601 UTC>"}]}]}
      Success: HTTP 200; тело ответа не разбирается
```

```text
repo: YMWB
path: price_updater_master.py
symbol: update_yandex
note: POST https://api.partner.market.yandex.ru/businesses/{businessId}/offer-prices/updates
      Auth: Bearer {ym_token}; Content-Type application/json
      Body: {"offers":[{"offerId":"<Sklad>","price":{"value":int,"currencyId":"RUR","discountBase":ceil(price*1.18/100)*100}}]}
      Success: HTTP 200
```

```text
repo: YMWB
path: order_notifications.py
symbol: get_orders_yandex_market
note: GET https://api.partner.market.yandex.ru/campaigns/{campaign_id}/orders
      Query: fake=False&status=PROCESSING&substatus=STARTED
      Auth: Bearer {ym_token}
      Parse: response.orders[]; поля id, delivery.shipments[].shipmentDate,
             items[].offerId|offerName|buyerPrice|count|subsidies[type=SUBSIDY].amount
```

Других вызовов `api.partner.market.yandex.ru` в репозитории нет.

## Оркестрация цикла

```text
repo: YMWB
path: services/marketplace_cycle.py
symbol: run_marketplace_cycle
note: Шаг 1 refresh_sklad → шаг 2 run_stock_updates (ym_update) →
      шаг 3 check_for_new_orders → шаг 4 update_all_prices (update_yandex).
      Интервал: APScheduler automation-cycle, default 5 мин.
```

```text
repo: YMWB
path: web_app.py
symbol: toggle_stock
note: OFF: send_zero_stocks_once("yandex") PUT нулей, затем backup Нал в temp_stock_backup.db
      и локальное обнуление. Флаг stock_flags.json["yandex"].
```

## Правила остатка и цены

```text
repo: YMWB
path: web_app.py
symbol: choose_best_supplier_for_row
note: Если Sklad.Наличие >= 1 — берём Sklad. Иначе min ОПТ среди Invask/Okno/United
      с nal>0; при равенстве цен порядок Invask > Okno > United.
```

```text
repo: YMWB
path: web_app.py
symbol: _calc_price
note: Цена = round((Опт + Опт * %/100) / 100) * 100
```

```text
repo: YMWB
path: auto_stock_updater.py
symbol: update
note: Внешний поставщик: Наличие < 3 → 0. Статус «выкл.» → Нал=0. Выключенный МП → Нал=0.
```

```text
repo: YMWB
path: order_notifications.py
symbol: update_stock
note: Списание: Sklad → Google Sheets КАЗНА/СКЛАД; иначе marketplace + !YMWB.db/prices.
      Если внешний поставщик и new_stock < 3 при stock>=3 → принудительно 0.
      UPDATE marketplace SET Нал WHERE Sklad=? — без фильтра Маркетплейс (кросс-МП эффект).
```

## Заказ → оператор

```text
repo: YMWB
path: order_notifications.py
symbol: notify_about_new_orders / write_order_to_gsheets
note: Дедуп System/order_ids.txt. Telegram. Лист КАЗНА/ЯМ: A="FBS" (хардкод),
      M="На сборке". Статус заказа в Partner API не обновляется.
```

## Чего нет в коде (отрицательные якоря)

- Нет POST/PUT заказов (accept, cancel, status).
- Нет labels / boxes / shipments / tracking.
- Нет returns / refunds.
- Нет catalog/offers mapping, cards, content.
- Нет webhooks / push-уведомлений ЯМ.
- Нет warehouse_id в payload остатков ЯМ (в отличие от Ozon).
- Нет 1С / PIM / MPS.
- Модуль `clients.http_client` импортируется, в shallow clone отсутствует.
