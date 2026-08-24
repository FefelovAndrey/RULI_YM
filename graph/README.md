# graph/ — граф проекта

Канон: **[graph.yaml](graph.yaml)**.  
Вид для человека: **[graph.md](graph.md)**.  
Типы узлов/рёбер: **[schema.md](schema.md)**.

## Как пользоваться

1. Перед Feature PRD — подграф: `depends_on`, `shares_rule`, `shares_entity`.
2. При изменении правила — править узел `R-*` / `D-*`, потом blast radius.
3. Не тащить в YAML все задачи Трекера — только сквозные F/R/D/E/C/W.

Обозримость слоя 1: см. [../meta/FOLDER-LOGIC.md](../meta/FOLDER-LOGIC.md).
