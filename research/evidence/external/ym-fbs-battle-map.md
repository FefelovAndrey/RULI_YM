# Карта боя: что Яндекс Маркет требует от FBS-продавца

Статус: `research`  
Дата: `2026-08-25`  
Модель в фокусе: **FBS** (товар у нас, доставку после приёмки делает Маркет)  
Не в фокусе: FBY, DBS, Экспресс, LaaS, Market Yandex Go

Это не AS IS RULI и не PRD. Это выжимка открытой документации Маркета: какие процессы вообще есть, чем они отличаются от «обычного магазина», какими методами API их закрывать. Следующий шаг — наложить эту карту на внутренние процессы.

Для агента канон процессов лежит рядом: [`ym-fbs-processes.yaml`](ym-fbs-processes.yaml). Здесь тот же смысл человеческим языком.

Источники — справка продавца и Partner API, снятые 2026-08-25. Методы живут, версии меняются: перед разработкой сверять страницу метода.

---

## Зачем эта карта

Волна 1 charter — первый заказ FBS end-to-end: карточка, цена, остаток, заказ, отгрузка, возврат. Чтобы не изобретать контур «как нам кажется», сначала фиксируем, **что Маркет считает процессом**. Потом смотрим, что из этого уже умеет WA / 1С / склад, а где дыра.

Автоматизация здесь почти везде = Partner API. Кабинет, Excel и YML остаются запасными путями и способом поднять пилот руками. Официальный модуль для 1С тоже есть — жив ли он у нас, скажет AS IS, не эта карта.

---

## Как устроен Маркет, пока не начали грузить товары

Два идентификатора, и их постоянно путают.

| Уровень | Параметр | Что это |
|---------|----------|---------|
| Кабинет | `businessId` | Один каталог на все магазины. Карточки живут здесь. |
| Магазин | `campaignId` | Кампания внутри кабинета. Заказы, отгрузки, цена «этого» магазина. |

Оба достаются одним запросом: [GET v2/campaigns](https://yandex.ru/dev/market/partner-api/doc/ru/reference/campaigns/getCampaigns.md).

Токен — **Api-Key**, заголовок `Api-Key`. Делает владелец или менеджер кабинета, до 30 штук, пока не удалят. Доступы режутся группами: карточки, цены, заказы, финансы, чаты. [Как создать](https://yandex.ru/dev/market/partner-api/doc/ru/concepts/api-key.md), [какие доступы](https://yandex.ru/dev/market/partner-api/doc/ru/concepts/access.md). Схема OAuth 2.0 в доке уже устаревшая.

База: `https://api.partner.market.yandex.ru`. Спека: [github.com/yandex-market/yandex-market-partner-api](https://github.com/yandex-market/yandex-market-partner-api). Индекс страниц: [llms.txt](https://yandex.ru/dev/market/partner-api/doc/ru/llms.txt).

Лимиты, о которых лучше знать сразу: не больше **четырёх параллельных** запросов на магазин или кабинет; тело запроса до **512 КБ**; пагинация через `pageToken` + `limit` (старые `page` / `pageSize` выводят). Перебор — HTTP `420`. Подробности: [ограничения](https://yandex.ru/dev/market/partner-api/doc/ru/concepts/limits.md).

Тестовые заказы не портят индекс качества и в обычных выборках не видны. Помечены `fake=true`. [Песочница](https://yandex.ru/dev/market/partner-api/doc/ru/concepts/sandbox.md).

---

## Картина целиком

```text
кабинет + токен + склад/график
        ↓
каталог (карточки) → контент категории → условия продажи в магазине
        ↓
цены (+ карантин)     остатки (доступно к заказу)
        ↓
заказ → сборка/коробки/КИЗ/ярлык → «Готов к отгрузке» → акт и first-mile
        ↓
невыкуп / возврат (решение за 48 часов)
        ↓
индекс качества следит за опозданиями и отменами
```

Отчёты продаж в этой цепочке не участвуют. Взаиморасчёты и претензии к Маркету — отдельные контуры: для первого заказа не блокер, но закрывающие и споры всё равно появятся, как только пойдут выплаты. См. P-YM-SETTLEMENT и P-YM-CLAIMS.

---

## Процессы

Идентификаторы `P-YM-*` совпадают с YAML. Кратко: специфика Маркета, затем методы со ссылками без разбора полей.

### P-YM-AUTH — доступ и идентификаторы

Без этого остальные методы не вызываются. Токен с нужными scope, `businessId`, `campaignId`, настройки кабинета (там же флаг `onlyDefaultPrice` — от него зависит, какой метод цен брать).

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `GET v2/campaigns` | Список магазинов, оба id | [getCampaigns](https://yandex.ru/dev/market/partner-api/doc/ru/reference/campaigns/getCampaigns.md) |
| `GET v2/campaigns/{campaignId}` | Карточка магазина | [getCampaign](https://yandex.ru/dev/market/partner-api/doc/ru/reference/campaigns/getCampaign.md) |
| `POST v2/businesses/{businessId}/settings` | Настройки кабинета | [getBusinessSettings](https://yandex.ru/dev/market/partner-api/doc/ru/reference/businesses/getBusinessSettings.md) |
| `GET v2/campaigns/{campaignId}/settings` | Настройки магазина | [getCampaignSettings](https://yandex.ru/dev/market/partner-api/doc/ru/reference/campaigns/getCampaignSettings.md) |
| `POST v2/auth/token` | Что умеет этот токен | [getAuthTokenInfo](https://yandex.ru/dev/market/partner-api/doc/ru/reference/auth/getAuthTokenInfo.md) |

Наложение на нас: кто хранит токен; один магазин или несколько `campaignId`; что осталось от старой интеграции.

---

### P-YM-WAREHOUSE — склад и график отгрузок

FBS для Маркета — это не «мы сами довезём покупателю». Мы собираем заказ и сдаём его **пачкой раз в день** в сортировочный центр, ПВЗ или курьеру Маркета, который приехал на склад.

Адрес склада в кабинете — реальный, даже если это угол в цехе. Точку приёма выбирают при подключении. График: отгружать в день заказа или на следующий. Заказы в выходной копятся до ближайшего рабочего дня. Крупногабарит (тяжелее 15 кг или длиннее 70 см по стороне) принимает не каждая точка.

API здесь закрывает не всё: склад и слот по-прежнему настраивают в кабинете. [Подключение FBS](https://yandex.ru/support/marketplace/ru/start/models/fbs).

Если в кабинете есть **группы складов**, меняется метод остатков: остаток пишут на один склад группы, остальные подтягиваются сами.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v3/businesses/{businessId}/warehouses` | Склады, если групп нет | [getPartnerWarehouses](https://yandex.ru/dev/market/partner-api/doc/ru/reference/warehouses/getPartnerWarehouses.md) |
| `POST v2/businesses/{businessId}/warehouses` | Склады, группы, статусы | [getPagedWarehouses](https://yandex.ru/dev/market/partner-api/doc/ru/reference/warehouses/getPagedWarehouses.md) |
| `POST v3/businesses/{businessId}/warehouse/models/status` | Выключить/включить FBS на складе | [updateWarehouseModelStatus](https://yandex.ru/dev/market/partner-api/doc/ru/reference/warehouses/updateWarehouseModelStatus.md) |

Пошагово: [склады](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/warehouses.md). После выключения модели товары сходят с витрины примерно за 15 минут. Если модель выключена дольше 30 дней — возврат до 4 часов. Если её выключил сам Маркет, этим методом обратно не включить.

---

### P-YM-PUSH — как узнаём о заказе

Два пути. Маркет рекомендует **API-уведомления**: он стучится на наш HTTPS. Иначе опрашиваем заказы. Для FBS в доке прямо сказано: не реже **раза в час**. Экспресс — каждые 10–15 минут, нам это не нужно, но порядок величины понятен.

Наш endpoint: только HTTPS, сертификат нормального УЦ, не самоподпись. Имеет смысл фильтровать IP Маркета. При подключении прилетает `PING` — ответить `200` за одну секунду, иначе интеграция не встанет. На обычное событие дают 10 секунд. Если молчим, Маркет ретраит и через 14 дней отключает уведомления (продажи при этом не останавливает).

Тестовые заказы в уведомления тоже приходят. Если повесить одно и то же на кабинет и на магазин, уйдёт одно письмо — на URL магазина.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST notification` | Входящее событие (наш URL) | [sendNotification](https://yandex.ru/dev/market/partner-api/doc/ru/push-notifications/reference/sendNotification.md) |
| `POST v1/businesses/{businessId}/orders` | Забрать тело заказа после события | [getBusinessOrders](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/getBusinessOrders.md) |

Как подключить: [уведомления](https://yandex.ru/dev/market/partner-api/doc/ru/push-notifications/index.md). Сравнение с опросом: [получение заказов](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/orders-receive.md).

События, которые нас касаются на e2e: новый заказ, изменение, статус, заявка на отмену, отмена, невыкуп/возврат и его статус. Чаты, споры, отзывы — позже, если спор по возврату не припрёт раньше.

Наложение: push или опрос — отдельный decision. И есть ли у WA публичный HTTPS.

---

### P-YM-CATALOG — карточки

SKU продавца на Маркете = `offerId`. Он уникален в кабинете. Каталог общий, не «на каждый магазин свой».

Маркет сам клеит оффер к карточке и может перепривязать, даже если передали желаемый `marketSku`. Категория — только **листовая**. Характеристики разные по категориям и Маркет их иногда дополняет.

Штрихкод производителя желателен; нет — генерируют свой. Не вся номенклатура продаётся на Маркете и не вся по FBS — перед пилотом свериться с [ограничениями ассортимента](https://yandex.ru/support/marketplace/ru/assortment/restrictions/index.md).

Загрузить товар в каталог ≠ выставить на витрину. Витрина — следующий процесс.

Пошагово: [добавление товаров](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/assortment-add-goods.md).

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/categories/tree` | Дерево категорий | [getCategoriesTree](https://yandex.ru/dev/market/partner-api/doc/ru/reference/categories/getCategoriesTree.md) |
| `POST v2/category/{categoryId}/parameters` | Характеристики листовой категории | [getCategoryContentParameters](https://yandex.ru/dev/market/partner-api/doc/ru/reference/content/getCategoryContentParameters.md) |
| `POST v2/businesses/{businessId}/offer-mappings/update` | Создать/обновить товары | [updateOfferMappings](https://yandex.ru/dev/market/partner-api/doc/ru/reference/business-offer-mappings/updateOfferMappings.md) |
| `POST v1/businesses/{businessId}/offer-mappings/barcodes/generate` | Штрихкоды Маркета | [generateOfferBarcodes](https://yandex.ru/dev/market/partner-api/doc/ru/reference/business-offer-mappings/generateOfferBarcodes.md) |
| `POST v2/businesses/{businessId}/offer-mappings` | Что в каталоге и к какой карточке привязано | [getOfferMappings](https://yandex.ru/dev/market/partner-api/doc/ru/reference/business-offer-mappings/getOfferMappings.md) |
| `POST v2/businesses/{businessId}/offer-mappings/delete` | Удалить из каталога | [deleteOffers](https://yandex.ru/dev/market/partner-api/doc/ru/reference/business-offer-mappings/deleteOffers.md) |
| `POST …/archive` / `…/unarchive` | Архив | [в архив](https://yandex.ru/dev/market/partner-api/doc/ru/reference/business-offer-mappings/addOffersToArchive.md) · [из архива](https://yandex.ru/dev/market/partner-api/doc/ru/reference/business-offer-mappings/deleteOffersFromArchive.md) |

Наложение: PIM или WA как источник; маппинг атрибутов; нельзя ли менять `offerId` как попало.

---

### P-YM-CONTENT — наполнение карточки

Основу (название, фото, цена кабинета, категория) несёт `offer-mappings/update`. Категорийные характеристики — отдельный контур `offer-cards`. Пустое значение затирает то, что уже стояло: передавать полный набор.

Пошагово: [изменение характеристик](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/content-change.md). Рекомендации Маркета по карточкам: [recommendations](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/recommendations.md).

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/businesses/{businessId}/offer-cards` | Статус заполненности и советы | [getOfferCardsContentStatus](https://yandex.ru/dev/market/partner-api/doc/ru/reference/content/getOfferCardsContentStatus.md) |
| `POST v2/businesses/{businessId}/offer-cards/update` | Записать характеристики | [updateOfferContent](https://yandex.ru/dev/market/partner-api/doc/ru/reference/content/updateOfferContent.md) |

---

### P-YM-PLACEMENT — витрина магазина

После каталога задают условия продажи в **магазине**: НДС, квант, минимальный объём. Только после этого товар имеет шанс появиться на витрине. Статусы карточки на витрине Маркет объясняет [в справке](https://yandex.ru/support/marketplace/assortment/add/statuses.html).

Скрыть с витрины, не убивая карточку, — `hidden-offers`. Удалить из ассортимента магазина — `offers/delete`. Это не то же самое, что удалить из каталога кабинета.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/campaigns/{campaignId}/offers/update` | Условия продажи | [updateCampaignOffers](https://yandex.ru/dev/market/partner-api/doc/ru/reference/offers/updateCampaignOffers.md) |
| `POST v2/campaigns/{campaignId}/offers` | Что в магазине и какие статусы | [getCampaignOffers](https://yandex.ru/dev/market/partner-api/doc/ru/reference/offers/getCampaignOffers.md) |
| `POST v2/campaigns/{campaignId}/offers/delete` | Убрать из магазина | [deleteCampaignOffers](https://yandex.ru/dev/market/partner-api/doc/ru/reference/offers/deleteCampaignOffers.md) |
| `GET/POST …/hidden-offers` (+ `…/delete`) | Скрыть / вернуть | [get](https://yandex.ru/dev/market/partner-api/doc/ru/reference/hidden-offers/getHiddenOffers.md) · [скрыть](https://yandex.ru/dev/market/partner-api/doc/ru/reference/hidden-offers/addHiddenOffers.md) · [вернуть](https://yandex.ru/dev/market/partner-api/doc/ru/reference/hidden-offers/deleteHiddenOffers.md) |

---

### P-YM-PRICE — цены и карантин

Либо одна цена на все магазины кабинета, либо своя на `campaignId`. Что можно — смотреть `onlyDefaultPrice` в настройках кабинета.

Резкий обвал относительно своей старой цены (по умолчанию больше чем вдвое) или относительно рынка кладёт товар в **карантин**. По новой цене он не продаётся, пока цену не подтвердят. Порог «относительно себя» крутится в кабинете; порог «относительно рынка» Маркет считает сам. [Цены в справке](https://yandex.ru/support/marketplace/ru/assortment/operations/prices).

НДС в метод цен не кладут — он в условиях продажи. Перед выгрузкой полезен калькулятор услуг: иначе «красивая» цена после комиссии уходит в минус.

Скидка на витрине: старая цена + новая, диапазон скидки 5–99%.

Пошагово: [изменение цен](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/assortment-change-prices.md).

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/businesses/{businessId}/offer-prices/updates` | Цена на все магазины | [updateBusinessPrices](https://yandex.ru/dev/market/partner-api/doc/ru/reference/prices/updateBusinessPrices.md) |
| `POST v2/campaigns/{campaignId}/offer-prices/updates` | Цена магазина | [updatePrices](https://yandex.ru/dev/market/partner-api/doc/ru/reference/prices/updatePrices.md) |
| `POST v2/businesses/{businessId}/offer-prices` | Прочитать цены кабинета | [getDefaultPrices](https://yandex.ru/dev/market/partner-api/doc/ru/reference/prices/getDefaultPrices.md) |
| `POST v2/campaigns/{campaignId}/offer-prices` | Прочитать цены магазина | [getPricesByOfferIds](https://yandex.ru/dev/market/partner-api/doc/ru/reference/prices/getPricesByOfferIds.md) |
| `POST …/price-quarantine` (+ `…/confirm`) | Карантин кабинета / магазина | [кабинет](https://yandex.ru/dev/market/partner-api/doc/ru/reference/price-quarantine/getBusinessQuarantineOffers.md) · [подтвердить](https://yandex.ru/dev/market/partner-api/doc/ru/reference/price-quarantine/confirmBusinessPrices.md) · [магазин](https://yandex.ru/dev/market/partner-api/doc/ru/reference/price-quarantine/getCampaignQuarantineOffers.md) · [подтвердить](https://yandex.ru/dev/market/partner-api/doc/ru/reference/price-quarantine/confirmCampaignPrices.md) |
| `POST v2/tariffs/calculate` | Сколько снимет Маркет | [calculateTariffs](https://yandex.ru/dev/market/partner-api/doc/ru/reference/tariffs/calculateTariffs.md) |
| `POST v2/businesses/{businessId}/offers/recommendations` | Рекомендации по цене | [getOfferRecommendations](https://yandex.ru/dev/market/partner-api/doc/ru/reference/offers/getOfferRecommendations.md) |

Акции Маркета — отдельный процесс P-YM-PROMO, в волну 1 charter не входит.

---

### P-YM-STOCK — остатки

Число, которое ждут, — **сколько ещё можно продать на Маркете**, не «сколько коробок на полке». Продажи с того же склада на Ozon, в розницу и куда угодно ещё должны быть уже вычтены. Иначе получим заказ, которого нет, и потом SHOP_FAILED.

Заказ на самом Маркете он резервирует сам. Наш фид после такого заказа должен отдать уже уменьшенное число.

Особый косяк FBS: заказ отменили **после** статуса «Готов к отгрузке». Маркет остаток на витрину сам не вернёт. Распаковать, прибавить, передать заново.

На витрине цифра догоняет передачу до 15 минут. Раз в сутки — нижняя планка из справки; при живых продажах так жить опасно.

Какой метод звать — зависит от групп складов. [Инструкция API](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/stocks.md), [справка](https://yandex.ru/support/marketplace/ru/assortment/operations/stocks).

| Метод | Когда | Ссылка |
|-------|--------|--------|
| `POST v3/businesses/{businessId}/offers/stocks/update` | Нет групп складов | [updateStocksOnPartnerWarehouses](https://yandex.ru/dev/market/partner-api/doc/ru/reference/stocks/updateStocksOnPartnerWarehouses.md) |
| `POST v3/businesses/{businessId}/offers/stocks` | Прочитать то же | [getStocksOnPartnerWarehouses](https://yandex.ru/dev/market/partner-api/doc/ru/reference/stocks/getStocksOnPartnerWarehouses.md) |
| `PUT v2/campaigns/{campaignId}/offers/stocks` | Есть группы складов | [updateStocks](https://yandex.ru/dev/market/partner-api/doc/ru/reference/stocks/updateStocks.md) |
| `POST v2/campaigns/{campaignId}/offers/stocks` | Остатки/оборачиваемость магазина | [getStocks](https://yandex.ru/dev/market/partner-api/doc/ru/reference/stocks/getStocks.md) |

API, Excel и YML можно мешать. Для боя это скорее риск рассинхрона, чем удобство.

---

### P-YM-ORDER-IN — заказ пришёл

В кабинете заказ сидит во вкладке «Ждут сборки», статус «Можно обрабатывать», на нём плановая дата отгрузки. Сначала сверка наличия. Нет товара — сразу в P-YM-ORDER-FAIL, не в сборку.

Рекомендуемый метод выборки — заказы **кабинета** `POST v1/businesses/{businessId}/orders`. Есть и магазинные GET по `campaignId`.

Заказы от организаций живут иначе: до пяти рабочих дней «Ожидает оплаты», товар лучше отложить, состав резать нельзя. Первый отменённый такой заказ снимает B2B-витрину, пока поддержка не разберётся. Нужен ли B2B на волне 1 — вопрос к наложению, не к Маркету.

Внешний номер (наш id в 1С/WA) можно отдать, пока заказ в `PROCESSING` / `STARTED`.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v1/businesses/{businessId}/orders` | Список и детали | [getBusinessOrders](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/getBusinessOrders.md) |
| `GET v2/campaigns/{campaignId}/orders` | Заказы магазина | [getOrders](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/getOrders.md) |
| `GET v2/campaigns/{campaignId}/orders/{orderId}` | Один заказ | [getOrder](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/getOrder.md) |
| `POST …/orders/{orderId}/external-id` | Наш номер | [updateExternalOrderId](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/updateExternalOrderId.md) |
| `POST …/business-buyer` и `…/documents` | Покупатель-юрлицо и документы | [buyer](https://yandex.ru/dev/market/partner-api/doc/ru/reference/order-business-information/getOrderBusinessBuyerInfo.md) · [docs](https://yandex.ru/dev/market/partner-api/doc/ru/reference/order-business-information/getOrderBusinessDocumentsInfo.md) |

Справка по ручной обработке: [как обрабатывать FBS](https://yandex.ru/support/marketplace/ru/orders/fbs/process). API: [FBS-заказы](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/fbs.md).

---

### P-YM-ORDER-PACK — собрали и сказали «готово»

Цепочка из доки:

1. Лист сборки — отчёт, не мгновенный файл: generate, потом `GET v2/reports/info/{reportId}`.
2. Коробки и коды маркировки — один `PUT …/boxes`. Пока нет `READY_TO_SHIP`, можно перезаписывать целиком.
3. Ярлык Маркета на каждое грузоместо. Старые наклейки снять. Три заказа с нечитаемым ШК — техническое отключение.
4. Статус `PROCESSING` + `READY_TO_SHIP`. Без него заказ не попадёт в акт.

Про Честный знак в API написано двусмысленно: для физлиц маркировка «необязательна», но ювелирка и ряд категорий требуют коды, и Маркет их проверяет. На возврате с КИЗ товар нужно вводить в оборот заново. Это место для сверки с нашим контуром ЧЗ, не для веры одной фразе.

Упаковка — [правила](https://yandex.ru/support/marketplace/ru/orders/fbs/packaging/rules). Ярлыки — [справка](https://yandex.ru/support/marketplace/ru/orders/fbs/packaging/marking). Сборку Маркет советует снимать на видео: при споре пригодится.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/reports/documents/shipment-list/generate` | Лист сборки | [generateShipmentListDocumentReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateShipmentListDocumentReport.md) |
| `GET v2/reports/info/{reportId}` | Забрать файл отчёта | [getReportInfo](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/getReportInfo.md) |
| `PUT v2/campaigns/{campaignId}/orders/{orderId}/boxes` | Коробки + КИЗ | [setOrderBoxLayout](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/setOrderBoxLayout.md) |
| `POST …/identifiers/status` | Проверка кодов | [getOrderIdentifiersStatus](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/getOrderIdentifiersStatus.md) |
| `PUT …/orders/{orderId}/identifiers` | Коды отдельным методом | [provideOrderItemIdentifiers](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/provideOrderItemIdentifiers.md) |
| `GET …/delivery/labels` | Ярлыки заказа | [generateOrderLabels](https://yandex.ru/dev/market/partner-api/doc/ru/reference/order-labels/generateOrderLabels.md) |
| `GET …/boxes/{boxId}/label` | Ярлык коробки | [generateOrderLabel](https://yandex.ru/dev/market/partner-api/doc/ru/reference/order-labels/generateOrderLabel.md) |
| `POST v2/reports/documents/labels/generate` | Ярлыки пачкой | [generateMassOrderLabelsReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateMassOrderLabelsReport.md) |
| `GET …/labels/data` | Данные, если печатаем сами | [getOrderLabelsData](https://yandex.ru/dev/market/partner-api/doc/ru/reference/order-labels/getOrderLabelsData.md) |
| `PUT …/orders/{orderId}/status` | Статус одного | [updateOrderStatus](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/updateOrderStatus.md) |
| `POST …/orders/status-update` | Статусы пачкой | [updateOrderStatuses](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/updateOrderStatuses.md) |

---

### P-YM-SHIPMENT — отвезли Маркету

СЦ и ПВЗ принимают не заказ, а **отгрузку** — пачку за день. Акт один на пачку.

Через API акт и подтверждение можно снять, только когда все заказы в отгрузке уже `READY_TO_SHIP` или отменены. Иначе ошибка со списком неготовых. В кабинете «подписать акт» — отдельная кнопка; в API подтверждение отгрузки — обязательный шаг, которого в UI нет.

Электронный акт имеет ту же силу, что бумажный, и три года лежит в кабинете. Бумажный шаблон — если едете не в свой слот или электронный не собрался.

Перенести заказ на завтра можно не позже чем за полчаса до отсечки. Позже — только отмена. И то и другое портит индекс.

Доверительная приёмка (без пересчёта в СЦ): число мест + ярлыки палет.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `GET …/first-mile/shipments/{shipmentId}` | Одна отгрузка | [getShipment](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/getShipment.md) |
| `PUT …/first-mile/shipments` | Несколько | [searchShipments](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/searchShipments.md) |
| `POST …/shipments/{shipmentId}/confirm` | Подтвердить | [confirmShipment](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/confirmShipment.md) |
| `GET …/shipments/reception-transfer-act` | Ближайшая отгрузка + акт | [downloadShipmentReceptionTransferAct](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/downloadShipmentReceptionTransferAct.md) |
| `GET …/act` · `discrepancy-act` · `inbound-act` · `transportation-waybill` | Документы | [акт](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/downloadShipmentAct.md) · [расхождения](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/downloadShipmentDiscrepancyAct.md) · [факт](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/downloadShipmentInboundAct.md) · [ТН](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/downloadShipmentTransportationWaybill.md) |
| `POST …/orders/transfer` | Перенос в следующую | [transferOrdersFromShipment](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/transferOrdersFromShipment.md) |
| `PUT …/pallets` + `GET …/pallet/labels` | Доверительная приёмка | [места](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/setShipmentPalletsCount.md) · [ярлыки](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/downloadShipmentPalletLabels.md) |
| `GET …/shipments/{shipmentId}/orders/info` | Можно ли печатать ярлыки | [getShipmentOrdersInfo](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/getShipmentOrdersInfo.md) |

Обзор FBS-методов скопом: [overview/fbs](https://yandex.ru/dev/market/partner-api/doc/ru/overview/fbs.md).

---

### P-YM-ORDER-FAIL — не смогли отгрузить как надо

Три действия, все бьют по индексу:

- отменить целиком: `CANCELLED` + `SHOP_FAILED`, обратно не вернуть;
- выкинуть позицию: снова `PUT …/boxes` (не любой набор имеет смысл для покупателя);
- перенести в следующую отгрузку.

Метод `PUT …/cancellation/accept` в доке — про DBS, когда покупатель отменяет уже в доставке. На FBS его в боевую цепочку не тащить.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `PUT …/orders/{orderId}/status` | Полная отмена продавца | [updateOrderStatus](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/updateOrderStatus.md) |
| `PUT …/orders/{orderId}/boxes` | Урезать состав | [setOrderBoxLayout](https://yandex.ru/dev/market/partner-api/doc/ru/reference/orders/setOrderBoxLayout.md) |
| `POST …/orders/transfer` | Следующая отгрузка | [transferOrdersFromShipment](https://yandex.ru/dev/market/partner-api/doc/ru/reference/shipments/transferOrdersFromShipment.md) |

---

### P-YM-RETURN — невыкуп и возврат

Покупатель не забрал или отказался до получения — невыкуп, товар едет обратно к нам. Уже получил и хочет вернуть — возврат. В API это один контур.

Решение по заявлению — **48 часов**. Сначала спрашиваем, какие решения Маркет вообще даёт, потом submit. Прозевали — Маркет может подтвердить возврат сам. Покупатель не согласен — спор и чат с арбитром.

По одному заказу возвратов бывает несколько. Частичный невыкуп оставляет заказ в `DELIVERED`: смотреть разницу `items` заказа и `items` возврата.

Деньги: сумма в `getBusinessOrders` возвраты не вычитает. Для расчёта вычитать `amount` возврата.

Пошагово: [невыкупы и возвраты](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/returns.md). Статусы FBS: [модель статусов](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/fby-fbs-express-return-status-model.md).

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `GET v2/campaigns/{campaignId}/returns` | Список | [getReturns](https://yandex.ru/dev/market/partner-api/doc/ru/reference/returns/getReturns.md) |
| `GET …/orders/{orderId}/returns/{returnId}` | Один | [getReturn](https://yandex.ru/dev/market/partner-api/doc/ru/reference/returns/getReturn.md) |
| `GET …/application` | Заявление | [getReturnApplication](https://yandex.ru/dev/market/partner-api/doc/ru/reference/returns/getReturnApplication.md) |
| `GET …/image/{imageHash}` | Фото | [getReturnPhoto](https://yandex.ru/dev/market/partner-api/doc/ru/reference/returns/getReturnPhoto.md) |
| `POST v1/businesses/{businessId}/returns/decisions` | Какие решения доступны | [getReturnAvailableDecisions](https://yandex.ru/dev/market/partner-api/doc/ru/reference/returns/getReturnAvailableDecisions.md) |
| `POST …/decision/submit` | Отдать решение | [submitReturnDecision](https://yandex.ru/dev/market/partner-api/doc/ru/reference/returns/submitReturnDecision.md) |
| `POST v2/businesses/{businessId}/chats/new` | Чат (если надо) | [createChat](https://yandex.ru/dev/market/partner-api/doc/ru/reference/chats/createChat.md) |

Фильтр «нужно наше решение»: статус `PREMODERATION_DECISION_WAITING`.

По срокам решение в доках расходится. В API — **48 часов** после создания заявления. В справке FBS по кабинету — **24 часа** с принятия заявки (округляется до часа вперёд), плюс до 72 часов на чат с покупателем. Если прозевали — Маркет сам одобряет возврат денег, оспорить это нельзя. Перед регламентом склада сверить, какой таймер реально горит в кабинете.

---

### P-YM-SETTLEMENT — взаиморасчёты и закрывающие

Маркет работает как **агент**: принимает деньги покупателя, удерживает своё, остаток переводит нам. Счёт на оплату услуг он не выставляет. «Отчёт комиссионера» с других площадок здесь называется иначе: **отчёт об исполнении поручения и о зачёте взаимных требований** (отчёт агента). В пакете закрывающих прямо написано, что это аналог отчёта комиссионера.

Пакет за календарный месяц (договор на размещение), первые семь рабочих дней следующего месяца, на почту / ЭДО / кабинет **Финансы → Закрывающие документы**:

| Документ | Зачем |
|----------|--------|
| Акт об оказании услуг и исполнении поручения | Что Маркет начислил за услуги |
| Счёт-фактура | То же для учёта НДС |
| Сводный отчёт по статистике | Сколько ушло в доставку, доставлено, не выкуплено, возвращено (FBY и FBS) |
| Отчёт агента | Сколько взяли с покупателей, сколько удержали, сколько перевели, сальдо |

[Пакет документов](https://yandex.ru/support/marketplace/ru/accounting/acts/main/). [Как читать отчёт агента](https://yandex.ru/support/marketplace/ru/accounting/acts/main/agent).

Выплаты — не «в конце месяца одной суммой». График в кабинете: каждый день (через день после **доставки**) или раз в неделю с отсрочкой 1 / 2 / 4 недели, даты 4 / 12 / 20 / 28. С 1 марта 2026 график и тариф перевода платежей менялись автоматически, если магазин сам не выбрал новый. [Выплаты](https://yandex.ru/support/marketplace/ru/accounting/payment).

Удержание услуг — взаимозачёт из платежей покупателей:

- **основные** (витрина, приём платежа, средняя миля, доставка покупателю…) — из платежа по этому же заказу;
- **дополнительные** (обработка заказа в СЦ, хранение невыкупов/возвратов, буст показов, полки…) — пачкой в начале следующего месяца.

Если денег покупателей не хватает, долг едет в следующие выплаты. Через 15 дней — предупреждение в кабинете, можно гасить QR или по реквизитам на счёт `ЛСМ-…` из акта, строго от того же юрлица. Старый режим «все услуги в следующем месяце» для магазинов до июля 2024: на него нельзя перейти, с него можно только уйти, и обратно нельзя. [Оплата услуг](https://yandex.ru/support/marketplace/ru/accounting/mutual-settlement).

Акт сверки — в кабинете, не раньше **10 числа** следующего месяца. Два вида: по услугам и по платежам. Подпись Маркета — бумагой на Новинский, 8.

Скидки за счёт Маркета приходят **баллами**, не живыми деньгами, и сами списываются в услуги. Компенсация за потерянный Маркетом товар в отчёте агента — не выручка, налогом не облагается (так пишет справка).

Нюансы для 1С:

- «подлежит перечислению» ≠ «уже на расчётном счёте»: график выплат даёт исходящее сальдо;
- услуги прошлого месяца в отчёте агента текущего;
- одно списание услуг по заказу может размазаться по нескольким платёжкам;
- ссылка на файл отчёта из API живёт 60 минут — качать сразу.

Через API закрывающие и детализация — ZIP/Excel, pretension workflow туда не входит. Scope токена: `finance-and-accounting`.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/reports/closure-documents/generate` | ZIP закрывающих за месяц (акт, СФ, статистика, отчёт агента) | [generateClosureDocumentsReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateClosureDocumentsReport.md) |
| `POST v2/reports/closure-documents/detalization/generate` | Схождение с закрывающими | [generateClosureDocumentsDetalizationReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateClosureDocumentsDetalizationReport.md) |
| `POST v2/reports/united-netting/generate` | Отчёт по платежам / платёжке / баллам | [generateUnitedNettingReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateUnitedNettingReport.md) |
| `POST v2/reports/united-marketplace-services/generate` | Стоимость услуг (по дате начисления или по дате акта) | [generateUnitedMarketplaceServicesReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateUnitedMarketplaceServicesReport.md) |
| `POST v2/reports/goods-realization/generate` | Реализация (в т.ч. когда Маркет сам продаёт невыкуп) | [generateGoodsRealizationReport](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/generateGoodsRealizationReport.md) |
| `GET v2/reports/info/{reportId}` | Забрать файл | [getReportInfo](https://yandex.ru/dev/market/partner-api/doc/ru/reference/reports/getReportInfo.md) |

Кабинет: **Финансы → Выплаты**, **Финансовые отчёты**, **Закрывающие документы**. Договор: [приложение 4](https://yandex.ru/legal/marketplace_service_agreement/#index__12_4) (перевод денег и оплата услуг).

Наложение: кто в 1С разносит отчёт агента vs акт услуг; ЭДО или почта; какой график выплат в кабинете; как на Ozon уже сверяют комиссию.

---

### P-YM-CLAIMS — претензии и споры

Тут два разных контура, их легко склеить в один «претензионный».

**1. Спор с покупателем по возврату** (ещё не претензия к Маркету). Покупатель не согласен с отказом — арбитраж. Маркет заводит чат, арбитр 48 часов, переписку с вами покупатель не видит. Решение арбитра **окончательное**. Игнор вопросов арбитра = решение без вас. API: уведомления `ORDER_RETURN_STATUS_UPDATED` и `CHAT_ARBITRAGE_STARTED`, чаты, статус возврата `PREMODERATION_DISPUTE`. [Возвраты FBS](https://yandex.ru/support/marketplace/ru/orders/returns/fbs).

Отдельно гарантия: можно забрать товар на экспертизу. Решение и акт сервиса — **10 дней** после получения, иначе деньги уйдут покупателю. Если отказали и товар надо вернуть покупателю — за 5 дней поставить «передал в доставку», иначе Маркет снова вернёт деньги, товар останется у нас.

**2. Претензия к Маркету** — когда уже получили возврат/невыкуп и с товаром не то, либо потеряли в доставке, либо кривые начисления. **API на подачу претензии нет.** Только кабинет, и только **владелец кабинета или бизнеса**. Сотрудник WA с ролью контента это не сделает.

Одна претензия = один магазин, не больше 10 заказов или 10 SKU. Срок рассмотрения до **35 рабочих дней**. Подпись — ПЭП, КЭП не нужна. Статус «Нужен ваш ответ»: допматериалы за **2 рабочих дня**.

Для FBS из справки [Подать претензию](https://yandex.ru/support/marketplace/ru/communication/claims):

| Тип | Окно |
|-----|------|
| Потеря товара в доставке | 3 месяца с плановой даты доставки |
| Потеря возврата | 3 месяца с даты заявки на возврат |
| Брак без акта расхождений | 48 часов с акта приёмки невыкупа |
| Брак, если акт расхождений уже есть | 30 календарных дней с выдачи товара |
| Недостача | 48 часов с акта при получении возврата |
| Кража / подмена в возврате | 48 часов с акта приёмки возврата |
| Оспорить возврат денег (повреждён / «брак», а товар годный) | 30 календарных дней с даты выдачи возврата |
| Некорректные начисления | без срока |

Забрать возврат из СЦ/ПВЗ — по электронной заявке (с 28 апреля без бумаги в общем случае). На месте смотреть грузоместа. Расхождение снять в зоне съёмки, фото в заявку за **48 часов** — с этого момента 30 дней на претензию. Заявку на вывоз надо закрыть за 15 дней, иначе отменится. Возвратный акт приёмки сохранять: без него спор с поддержкой пустой.

Компенсация за порчу — 10–100% от **объявленной ценности** (FBS), не от последней цены продажи. Оспорить размер/отказ — ещё 30 дней с уведомления Маркета, плюс акт о повреждениях.

Несогласны с актом услуг — не претензия из списка выше, а обращение **Поддержка → Взаиморасчёты и документы**.

Договор: [приложение 4.1](https://yandex.ru/legal/marketplace_service_agreement/#index__12_41) (претензии покупателей), [приложение 5](https://yandex.ru/legal/marketplace_service_agreement/#index__12_5) (возврат FBS).

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| — | Подать претензию к Маркету | нет в Partner API; кабинет |
| `POST notification` | Спор, статус возврата | [sendNotification](https://yandex.ru/dev/market/partner-api/doc/ru/push-notifications/reference/sendNotification.md) |
| методы P-YM-RETURN | Решение по заявке покупателя | см. выше |
| `POST v2/businesses/{businessId}/chats/new` и история чата | Чат с покупателем / арбитром | [createChat](https://yandex.ru/dev/market/partner-api/doc/ru/reference/chats/createChat.md) · [getChatHistory](https://yandex.ru/dev/market/partner-api/doc/ru/reference/chats/getChatHistory.md) |
| `POST v2/businesses/{businessId}/chats` | Список чатов (в т.ч. арбитраж) | [getChats](https://yandex.ru/dev/market/partner-api/doc/ru/reference/chats/getChats.md) |

Наложение: кто забирает возвраты и фотографирует расхождения в 48 часов; кто владелец кабинета для претензии; как это уже сделано на Ozon; не смешивать клиентский возврат волны 1 с претензией к Маркету (в charter — отдельно согласовать).

---

### P-YM-QUALITY — индекс качества

Число от 0 до 100, видит только продавец. Для FBS появляется после 10 заказов за 30 дней.

Ниже 40 — через сутки магазин гасится на неделю. Чтобы держать индекс около 98: не больше 4% отгрузок не в слот и не больше 2% отмен по своей вине. Опоздание, перенос, КГТ не в ту дату — всё считается. Плата за отмену: от 150 ₽ до 5000 ₽ за товар, процент от индекса. Пока индекса нет, новым считают как 98–100.

Если за неделю 20% и больше отгрузок с опозданием, Маркет может сам растянуть график и отнять скидку за скорость.

| Метод | Зачем | Ссылка |
|-------|--------|--------|
| `POST v2/businesses/{businessId}/ratings/quality` | Индекс магазинов | [getQualityRatings](https://yandex.ru/dev/market/partner-api/doc/ru/reference/ratings/getQualityRatings.md) |
| `POST v2/campaigns/{campaignId}/ratings/quality/details` | Какие заказы качнули | [getQualityRatingDetails](https://yandex.ru/dev/market/partner-api/doc/ru/reference/ratings/getQualityRatingDetails.md) |

Справка: [индекс FBS](https://yandex.ru/support/marketplace/ru/quality/score/fbs). Техническое отключение за кривые ярлыки и массовые опоздания: [качество / tech](https://yandex.ru/support/marketplace/ru/quality/tech).

---

### Что сознательно отодвинули

| Процесс | Почему не в боевой e2e | Куда смотреть |
|---------|------------------------|---------------|
| P-YM-SETTLEMENT | В charter волны 1 не блокер первого заказа. Закрывающие всё равно придут. | раздел выше |
| P-YM-CLAIMS | Претензия к Маркету — отдельно от клиентского возврата. | раздел выше |
| P-YM-REPORT | Аналитика WA «как у Ozon» — волна 2. | [отчёты](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/reports.md) |
| P-YM-PROMO | Акции и буст. Не блокер первого заказа. | [акции](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/promos.md) |
| P-YM-CHAT | Поддержка как контур внедрения — out of scope. Чат нужен на споре возврата. | [чаты](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/chats.md) |
| P-YM-FBY | Поставки на склад Маркета. Не путать с FBS-отгрузкой. | [overview FBY](https://yandex.ru/dev/market/partner-api/doc/ru/overview/fby.md) |

Сравнение методов по моделям: [comparison](https://yandex.ru/dev/market/partner-api/doc/ru/overview/comparison.md). Общие методы кабинета: [overview/business](https://yandex.ru/dev/market/partner-api/doc/ru/overview/business.md).

---

## Другие способы, кроме API

Маркет явно говорит: ассортимент и остатки можно вести кабинетом, Excel, YML **параллельно** с API. Для склада с объёмом RULI это скорее источник рассинхрона.

Есть [модули CMS и партнёров](https://yandex.ru/support/marketplace/ru/tools/api-modules) и [модуль для 1С](https://yandex.ru/dev/market/partner-api/doc/ru/modules/1c.md). На AS IS стоит глянуть: не торчит ли модуль в 1С с прошлой жизни на Маркете.

---

## Как накладывать на внутренний AS IS

Идти по `happy_path` в YAML. Для каждого `P-YM-*` ответить:

1. Есть ли у нас такой процесс на Ozon / в рознице / в старом ЯМ?
2. Кто SoT (WA, 1С, PIM, Excel, человек)?
3. Какой метод из таблицы закроет дыру, а какой уже закрыт кабинетом и так можно оставить на пилот?
4. Где сломается индекс качества, если процесс ручной?

Открытые вопросы с этого шага — в [`briefs/open-questions.md`](../../briefs/open-questions.md). Reality Brief не заполнять, пока нет внутреннего AS IS: иначе смешаем «как требует Маркет» и «как у нас».

Связь с графом (черновик, не патчить `graph.yaml` с этой карты):

| Процесс ЯМ | Узел графа |
|------------|------------|
| P-YM-AUTH, P-YM-WAREHOUSE | C-YM-API |
| P-YM-CATALOG, P-YM-CONTENT, P-YM-PLACEMENT | F-CATALOG / E-OFFER |
| P-YM-PRICE | F-PRICE-SYNC |
| P-YM-STOCK | F-STOCK-SYNC |
| P-YM-PUSH … P-YM-ORDER-FAIL, P-YM-SHIPMENT | F-ORDERS / E-ORDER |
| P-YM-RETURN | F-RETURNS (клиентский возврат) |
| P-YM-SETTLEMENT | вне графа волны 1; charter: согласовать отдельно |
| P-YM-CLAIMS | F-RETURNS «претензии» сверх клиентского возврата — отдельно |

---

## Источники

Снято 2026-08-25, открытые страницы.

API:

- [Введение](https://yandex.ru/dev/market/partner-api/doc/ru/)
- [Обзор методов](https://yandex.ru/dev/market/partner-api/doc/ru/overview/index.md)
- [Методы FBS](https://yandex.ru/dev/market/partner-api/doc/ru/overview/fbs.md)
- [Методы кабинета](https://yandex.ru/dev/market/partner-api/doc/ru/overview/business.md)
- [Пошаговые инструкции](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/index.md)
- [OpenAPI на GitHub](https://github.com/yandex-market/yandex-market-partner-api)

Справка продавца:

- [Модель FBS](https://yandex.ru/support/marketplace/ru/start/models/fbs)
- [Обработка и отгрузка](https://yandex.ru/support/marketplace/ru/orders/fbs/process)
- [Остатки](https://yandex.ru/support/marketplace/ru/assortment/operations/stocks)
- [Цены и карантин](https://yandex.ru/support/marketplace/ru/assortment/operations/prices)
- [Индекс качества FBS](https://yandex.ru/support/marketplace/ru/quality/score/fbs)
- [API и модули](https://yandex.ru/support/marketplace/ru/tools/api-modules)
- [Документы по договору / отчёт агента](https://yandex.ru/support/marketplace/ru/accounting/acts/main/)
- [Выплаты](https://yandex.ru/support/marketplace/ru/accounting/payment)
- [Оплата услуг / взаимозачёт](https://yandex.ru/support/marketplace/ru/accounting/mutual-settlement)
- [Финансовые отчёты](https://yandex.ru/support/marketplace/ru/accounting/transactions)
- [Возвраты FBS](https://yandex.ru/support/marketplace/ru/orders/returns/fbs)
- [Подать претензию](https://yandex.ru/support/marketplace/ru/communication/claims)
- [Оферта, приложения 4 / 4.1 / 5](https://yandex.ru/legal/marketplace_service_agreement/)
