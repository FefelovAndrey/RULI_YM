# features/ — Feature PRD

Слой 3. Одна фича = папка `F-XXXX-slug/` + запись в [STATUS.md](STATUS.md).

Порядок проработки — по волнам графа (`graph/graph.yaml`), ключевые сначала.  
Не писать PRD при открытых `depends_on` → `D-*` без решения.

## Файлы

| Путь | Содержание |
|------|------------|
| [STATUS.md](STATUS.md) | Сводка статусов |
| [_template/PRD.md](_template/PRD.md) | Шаблон |
| [F-ORDERS/PRD.md](F-ORDERS/PRD.md) | FBS: приём / сборка / ярлык / SHIPPED (после прогона 2026-09-01) |
| `F-*/PRD.md` | Контракт фичи |

Логика: [../meta/FOLDER-LOGIC.md](../meta/FOLDER-LOGIC.md)
