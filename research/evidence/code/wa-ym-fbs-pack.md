# Якоря: FBS-сборка / ярлык / SHIPPED (Webasyst)

Код: `C:\Users\Дмитрий Ли\Projects\RULI\CodeBase\caroptics`  
Плагины: `wa-apps/shop/plugins/{ym,clabels,rulimpsupplies}`  
Прогон: [fbs-test-2026-09-01.md](../../briefs/fbs-test-2026-09-01.md)

Секреты и адреса покупателей сюда не копировать. OAuth/Api-Key — только имена переменных.

---

## Приёмка

| Что | Где |
|-----|-----|
| Callback accept | `shopYmPluginFrontendApi` — `action=/order/accept` |
| Создание заказа Shop | `shopYmApi::createOrder` |
| Дыра | **не вставляет** строку в `shop_ym_order_state` |
| Токен URL | hex из настроек плагина `ym`, не `y0_` |
| Формат URL | `/ym_api/{profile}/?action=/order/accept?auth-token=` (два `?`) |

Импорт 2026-09-01: Postman, тело с `"fake": false`; кабинет остался sandbox.

---

## Сборка и READY_TO_SHIP

| Что | Где |
|-----|-----|
| Пункт «Подбор» | `BackendNav.html` — **закомментирован** |
| Переход на упаковку | Shop workflow: `na-upakovku` только из `in_stock` (`selectionChangeStatus`) |
| UI Сборки | `?plugin=clabels&action=check` |
| Гейт ярлыка / info | `shopClabelsMarket::getOrderCheckInfo` — `is_label` |
| PUT READY_TO_SHIP | `shopClabelsPluginBackendMarketOrderShipping` |
| Камера | складское РМ; `is_testing=true` включает тестовую авторизацию |

Скан на Сборке: **номер Shop** (2534310), не номер Маркета.

PUT **09:52:51**: `…/campaigns/137514772/orders/61089913793/status.json` → `PROCESSING` + `READY_TO_SHIP`.

---

## Ярлык

| Что | Где |
|-----|-----|
| Скачать PDF (карточка YM) | `shopYmPluginBackendGetOrderLabels` — `?plugin=ym&action=getOrderLabels&order_id=` **id Маркета** |
| Печать QZ (Сборка) | `shopClabelsPluginBackendMarketPrintLabel` |
| Partner API | GET `…/orders/{id}/delivery/labels.json` |

Живой PDF: **12:07, 12:30, 12:33**. Кнопка Сборки не открылась из-за дыры `ym_order_state` / `is_label`.

---

## SHIPPED

| Что | Где |
|-----|-----|
| Кнопка карточки YM | «Заказ передан в доставку» → `ymChangeStatus` |
| Обработчик | `shopYmPluginBackendChangeOrderStatus` |
| Shop «Отправлен» | workflow action `ship` (не PUT SHIPPED; можно нажать повторно — нельзя) |

PUT **12:35:57** → `SHIPPED`. GET 12:36–12:40 подтвердили. Повтор 12:39 идемпотентен.

---

## First-mile

| Что | Где |
|-----|-----|
| Список отгрузок | `shopRulimpsuppliesPluginYmActions::loadShipmentsAction` |
| Склады | `rulimpsuppliesYmApi::getWarehouses` — **захардкоженный OAuth** (не цитировать; вынести) |

На fake-заказе список пустой при дефолтных фильтрах — ожидаемо для этого SKU; чужие акты не подтверждать.

---

## Клиент API

`shopYmApi`: OAuth + `.json` на path. На стенде 2026-09-01 ответы 200. Долг: Api-Key, URL без `.json`.

Лог: `C:\wamp64\logs\query.17.log`. Паттерн `URL … NULL` = второй dump пустого тела, не обрыв HTTP.
