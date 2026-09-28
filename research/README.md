# research/ — исследования → AS IS

Слой 1. Сюда — разведка и формулировка as-is **до** Feature PRD.

Порядок работы:

1. Собрать evidence в `evidence/`
2. Синтезировать `as-is/`
3. Зафиксировать `briefs/reality-brief.md` и открытые вопросы

## Файлы / папки

| Путь | Содержание |
|------|------------|
| [as-is/fbs-fulfillment-wa.md](as-is/fbs-fulfillment-wa.md) | **Канон AS IS исполнения FBS в WA** (сборка / ярлык / SHIPPED) |
| [briefs/ym-order-receive-wa.md](briefs/ym-order-receive-wa.md) | **Что доработать в WA, чтобы заказ ЯМ появился в Shop без Postman** |
| [briefs/ym-orders-import-cron-tz.md](briefs/ym-orders-import-cron-tz.md) | **ТЗ:** импорт заказов ЯМ в WA — включение cron (`cli_orders_states` + `shop ym 5`) |
| [briefs/fbs-test-2026-09-01.md](briefs/fbs-test-2026-09-01.md) | Прогон sandbox 2026-09-01 + **чеклист доработок** |
| [as-is/ym-catalog-price-stock-wa.md](as-is/ym-catalog-price-stock-wa.md) | Карточки / цены / остатки в плагине `ym` |
| [briefs/catalog-price-stock-test.md](briefs/catalog-price-stock-test.md) | Протокол теста витрины (SKU 1107932) |
| [briefs/ym-card-send-ozon-hook.md](briefs/ym-card-send-ozon-hook.md) | **Карточка ЯМ as-is и почему в процесс Ozon не встать** (срез кода 2026-09-24) |
| [briefs/ym-card-data-inflow.md](briefs/ym-card-data-inflow.md) | **Как поля карточки попадают в WA** (1С / PIM / очередь) и что выгружает на Маркет; схема по системам |
| [briefs/ym-vs-ozon-catalog-process.md](briefs/ym-vs-ozon-catalog-process.md) | Карточки/цены/остатки ЯМ vs боевой контур Ozon |
| [briefs/ym-catalog-process-ba.md](briefs/ym-catalog-process-ba.md) | БА: массовая выгрузка ЯМ «как Ozon», код с нуля, Ozon freeze |
| [briefs/ym-fbs-pricing.md](briefs/ym-fbs-pricing.md) | **Цена SKU FBS:** справочник удержаний 2026 + пример расчёта |
| [briefs/ym-fee-daily-sync.md](briefs/ym-fee-daily-sync.md) | Суточный sync FEE% → UI комиссий (UML + зонд `tariffs/calculate`) |
| [briefs/ym-fee-categories-tz.md](briefs/ym-fee-categories-tz.md) | **ТЗ разработчику:** процессы FEE API/парсинг, контракты, UML sequence |
| [briefs/ym-boost-tz.md](briefs/ym-boost-tz.md) | **ТЗ:** перенос `shopYmBusts` под UI ym-boost.html (CSV + % маржи) |
| [briefs/ym-boost-business.md](briefs/ym-boost-business.md) | Буст: описание для бизнеса (на базе ТЗ) |
| [briefs/ym-export-tz.md](briefs/ym-export-tz.md) | **ТЗ:** выгрузка цен под UI ym-export.html (F-PRICE-SYNC + business URL) |
| [briefs/ym-export-business.md](briefs/ym-export-business.md) | Выгрузка цен: описание для бизнеса (на базе ТЗ) |
| [briefs/ym-fbs-auto-parts-pricing.canvas.tsx](briefs/ym-fbs-auto-parts-pricing.canvas.tsx) | Canvas-калькулятор FBS автозапчасти (архив; живой — в Cursor canvases) |
| [briefs/ym-ozon-arch-pricing-example.canvas.tsx](briefs/ym-ozon-arch-pricing-example.canvas.tsx) | Пример расчёта ЯМ в архитектуре Ozon (+ `.data.json` снимок) |
| [briefs/page/ym-price-mgmt-ui-example.html](briefs/page/ym-price-mgmt-ui-example.html) | Полный прототип UI «Ценообразование ЯМ» (Pages: [пример](https://fefelovandrey.github.io/RULI_YM/ym-price-mgmt-ui-example.html)) |
| [briefs/ym-price-mgmt-tz.md](briefs/ym-price-mgmt-tz.md) | **ТЗ разработчику:** раздел ценообразования ЯМ (экран + движок P_min) |
| [briefs/ym-pricing-aligned-to-ozon-arch.canvas.tsx](briefs/ym-pricing-aligned-to-ozon-arch.canvas.tsx) | Схема: процесс как Ozon, тарифы ЯМ |
| [briefs/ym-profit-options.md](briefs/ym-profit-options.md) | Прибыль/удержания: какие методы API, не ставки для прайса |
| [as-is/ym-returns-wa.md](as-is/ym-returns-wa.md) | Невыкупы / возвраты: `ym` + clabels (код; GET `/returns` не снимали) |
| [briefs/returns-cancel-test-2026-09-03.md](briefs/returns-cancel-test-2026-09-03.md) | Прогон sandbox: accept → отмена + чеклист доработок |
| [briefs/returns-test.md](briefs/returns-test.md) | Чеклист чтения GET `/returns` |
| [as-is/](as-is/) | Прочее as-is (в т.ч. YMWB — вне скоупа) |
| [evidence/](evidence/) | Сырьё: code, db, tracker, external |
| [briefs/](briefs/) | Reality Brief, Open Questions |

Логика: [../meta/FOLDER-LOGIC.md](../meta/FOLDER-LOGIC.md)
