# F-ORDERS — заказы FBS (приём / сборка / ярлык / отгрузка)

Статус: `specified`  
Волна: `W2-CORE`  
Граф: см. `../../graph/graph.yaml` (id: `F-ORDERS`)  
Трекер: —  
Решение: [D-YM-FULFILLMENT](../../decisions/D-YM-FULFILLMENT.md)  
Прогон: [fbs-test-2026-09-01.md](../../research/briefs/fbs-test-2026-09-01.md)

## Context / Problem

As-is: в WA путь Partner API уже живой. 2026-09-01 на sandbox (Маркет `61089913793`, Shop `2534310`, профиль `17`) прошли accept → `na-upakovku` → Сборка → `READY_TO_SHIP` → PDF ярлыка → `SHIPPED`. First-mile не закрывали (`fake`).

To-be: тот же контур без Postman и без обхода ярлыка. Оператор на Сборке видит «Получить наклейку» после READY_TO_SHIP; `shop_ym_order_state` пишется с accept; чужие first-mile акты не подтверждаются.

Reality Brief: `../../research/briefs/reality-brief.md`. AS IS: `../../research/as-is/fbs-fulfillment-wa.md`.

## Scope

### In

- Запись `shop_ym_order_state` при `/order/accept`
- Гейт ярлыка на Сборке (`is_label`) по статусу Маркета и/или локальному state
- Операторский путь: `in_stock` → «На упаковку» → clabels; вкладка «Подбор» или явная альтернатива
- Одноразовость Shop `ship` («Отправлен») и кнопки «Заказ передан в доставку» после успеха
- Явный статус, если нет `id_1c` (не тихий skip 1С-гейта)
- Понятный ярлык: QZ на Сборке и/или PDF с карточки YM
- First-mile: дефолты списка отгрузок; только свой акт на живом FBS
- Авто-accept живого заказа без Postman (следующий прогон)

### Out

- Каталог / цены / остатки (`F-CATALOG`, `F-PRICE-SYNC`, `F-STOCK-SYNC`)
- YMWB
- Повтор заказа `61089913793` / `2534310`
- Подтверждение чужих first-mile отгрузок
- Смена `fake: true` в кабинете sandbox ради этого заказа
- Возвраты (`F-RETURNS`)

## Preconditions (depends_on)

| ID | Что должно быть правдой |
|----|-------------------------|
| `C-YM-API` | Профиль `ym` ходит в Partner API (на 2026-09-01 GET/PUT `.json` = HTTP 200) |
| `F-STOCK-SYNC` | Не блокер среза доработок: тестовый оффер уже был на витрине |
| `D-YM-FULFILLMENT` | Исполнение остаётся в WA, не выносится в новый сервис |

## Shared rules / entities

| ID | Как используем |
|----|----------------|
| `E-ORDER` | Два id: Маркет ≠ Shop. Сборка сканит Shop; ярлык PDF — id Маркета |
| `C-YM-API` | Клиент `shopYmApi` + вызовы clabels/rulimpsupplies |

## Business rules (локальные для фичи)

| ID | Правило | Источник |
|----|---------|----------|
| BR-1 | На `/order/accept` создаётся заказ Shop **и** строка `shop_ym_order_state` | код: `shopYmApi::createOrder` не пишет state; прогон 2026-09-01 |
| BR-2 | Кнопка «Получить наклейку» после `READY_TO_SHIP` (или живого PDF), без обязательного Shop `shipped` как единственного ключа | `shopClabelsMarket::getOrderCheckInfo` |
| BR-3 | `na-upakovku` только из `in_stock` | Shop workflow |
| BR-4 | Скан Сборки — номер Shop, не Маркета | clabels `action=check` |
| BR-5 | PUT `SHIPPED` идемпотентен; повторный клик после успеха не шлёт второй осмысленный сдвиг | GET 12:36–12:40, повтор 12:39 |
| BR-6 | Fake-заказ не кладётся в живые first-mile отгрузки | rulimpsupplies + кабинет `fake: true` |
| BR-7 | На бою `is_testing` камеры склада выключен | clabels |
| BR-8 | OAuth складов first-mile — из настроек профиля, не литерал в коде | `rulimpsuppliesYmApi::getWarehouses` |

## Flows

### Happy path

1. Маркет вызывает `POST /ym_api/{profile}/?action=/order/accept` (токен плагина, не `y0_`).
2. Shop: заказ + строка `ym_order_state`; статус, с которого доступна упаковка (`in_stock` → `na-upakovku`).
3. Оператор на Сборке сканирует **номер Shop**.
4. PUT `PROCESSING` + `READY_TO_SHIP`.
5. На Сборке доступна печать ярлыка (QZ и/или PDF).
6. Shop «Отправлен» и/или карточка YM «Заказ передан в доставку» → PUT `SHIPPED`.
7. First-mile (живой FBS): своя отгрузка, слот, акт — без чужих документов.

Прогон 2026-09-01 закрыл шаги 1 (Postman) и 3–6 (ярлык — PDF с карточки YM, не кнопка Сборки). Шаг 7 не делали.

### Fail / edge

| Кейс | Ожидание |
|------|----------|
| Accept без строки `ym_order_state` | Не должно остаться: последующий `updateByField` не бьёт в пустоту |
| READY_TO_SHIP есть, `is_label` false | Гейт смотрит Маркет/кэш PDF, кнопка Сборки открыта |
| Заказ уже `shipped`, повтор «Отправлен» | Действие скрыто или no-op с понятным сообщением |
| Нет `id_1c` | Видно «1С не привязана», не тихий skip |
| Повторная печать при `master_barcode` | Блок с объяснением, не молчаливый отказ |
| `query.17.log`: `NULL` после URL | Не трактовать как fail API (второй dump пустого тела) |
| Список rulimpsupplies пуст при дефолте «сегодня» и без `OUTBOUND_CREATED` | Подсказка или расширенный дефолт |
| Чужой акт first-mile на стенде | Не подтверждать |

## Data / contracts

| Поле / сущность | Смысл |
|-----------------|-------|
| id Маркета | Partner API `orders/{id}`; ярлык `getOrderLabels` |
| id Shop | `shop_order`; скан Сборки |
| `ym_shop_id` | профиль кампании (тест: 17 / campaign `137514772`) |
| `shop_ym_order_state` | локальный статус YM; обязан появиться на accept |
| Маркет status | `PROCESSING` + substatus `READY_TO_SHIP` → `SHIPPED` |
| Shop status | `in_stock` → `na-upakovku` → `shipped` |
| Короб | `{marketOrderId}-1` (тест: `61089913793-1`) |

Миграция: если таблица `shop_ym_order_state` уже есть — **insert при createOrder**, не новая схема. Backfill старых заказов — по необходимости, не этот sandbox-заказ.

## Systems touched

| Система | Что меняется |
|---------|--------------|
| WA `plugins/ym` | insert `ym_order_state` на accept; клиент API (P2: Api-Key / без `.json`) |
| WA `plugins/clabels` | гейт `is_label`; подпись печати; 1С-гейт; камера `is_testing` |
| WA Shop workflow | подпись/ACL `ship`; пункт «Подбор» |
| WA `plugins/rulimpsupplies` | дефолты списка; токен складов из настроек |
| Яндекс Маркет Partner API | без смены контракта методов; живой прогон без Postman |

Код: `refs/codebases.md` → WA / caroptics. Якоря: `research/evidence/code/wa-ym-fbs-pack.md`.

## Acceptance criteria

```gherkin
Feature: FBS заказ в WA без обходов

  Scenario: Accept пишет состояние YM
    Given callback /order/accept для профиля ym
    When заказ создан в Shop
    Then существует строка shop_ym_order_state для этого заказа
    And последующий updateByField обновляет её, а не 0 строк

  Scenario: Ярлык на Сборке после READY_TO_SHIP
    Given заказ в na-upakovku и Маркет READY_TO_SHIP
    When оператор открывает Сборку по номеру Shop
    Then is_label истинно
    And доступна «Получить наклейку» либо явно подписанное скачивание PDF

  Scenario: SHIPPED один раз
    Given Маркет уже SHIPPED
    When оператор снова жмёт «Заказ передан в доставку» или Shop «Отправлен»
    Then повторный сдвиг статуса не требуется
    And UI не предлагает опасный повтор как основной шаг

  Scenario: Авто-accept без Postman
    Given живой FBS заказ fake=false
    When Маркет шлёт accept на /ym_api/{profile}/
    Then заказ появляется в Shop без ручного Postman
```

Негативные AC:

```gherkin
  Scenario: Нет id_1c
    Given заказ ЯМ без привязки 1С
    When Сборка проверяет CheckResult
    Then оператор видит «1С не привязана»
    And гейт не глотается молча

  Scenario: First-mile не чужой
    Given на стенде чужие отгрузки Маркета
    Then UI не провоцирует confirm чужого акта
    And fake-заказ не обязан попасть в живой список отгрузок
```

Чеклист P0/P1/P2 и следующий живой тест — в прогоне, не дублировать здесь как второй SoT.

## Related / impact

- `depends_on`: `C-YM-API`, `F-STOCK-SYNC`, `D-YM-FULFILLMENT` (остатки — не срез доработок приёмки/ярлыка)
- `shares_entity`: `E-ORDER`
- `enables`: `C-YM-API` → `F-ORDERS`
- Blast radius: правка гейта ярлыка / accept затрагивает только контур заказа; карточки и остатки не патчить. `F-RETURNS` зависит от `F-ORDERS` — не стартовать, пока SHIPPED-путь не стабилен на живом FBS.

## Open questions

| ID | Вопрос |
|----|--------|
| Q-011 | Авто-accept живого FBS без Postman на профиле 17 стабилен? |
| Q-012 | First-mile на живом FBS: свой акт/слот, без чужих документов? |
| Q-002 | Только FBS или ещё FBY/DBS — вне этой фичи, пока нет схемы |

## Traceability

| Сигнал | Правило | Touchpoint |
|--------|---------|------------|
| Прогон 2026-09-01 | BR-1 | `shopYmApi::createOrder` |
| Кнопка ярлыка не открылась | BR-2 | `shopClabelsMarket::getOrderCheckInfo` |
| PUT 09:52 READY_TO_SHIP | happy path 4 | `shopClabelsPluginBackendMarketOrderShipping` |
| PDF 12:07–12:33 | happy path 5 | `shopYmPluginBackendGetOrderLabels` |
| PUT 12:35 SHIPPED | BR-5 | `shopYmPluginBackendChangeOrderStatus` |
| D-YM-FULFILLMENT A | scope In | WA plugins, не новый сервис |
