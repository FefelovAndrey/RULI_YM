# Router — фазы и гейты

## Фазы

| Фаза | Кто | Выход | Папка |
|------|-----|-------|--------|
| 0 Intake | Clarifier + человек | бриф, вопросы | `research/briefs/` + при необходимости Трекер |
| 1 Разведка | Process / Code / Data / Integration | evidence + as-is | `research/` |
| 2 Alignment | Critic + человек | согласие по outcome | вопросы в open-questions |
| 3 Framing | Solution Framer | 2–3 варианта → выбор | `decisions/` |
| 4 Graph update | Graph Steward | узлы/рёбра | `graph/` |
| 5 Spec | Requirements Author | Feature PRD | `features/F-*/` |
| 6 Audit | Spec Auditor | гейт AC / согласованность | комментарии к PRD |
| 7 Dual review | заказчик + разработка | go | Трекер / комитет |

## Гейты (стоп, если красный)

1. Problem — outcome проверяем  
2. Reality — есть Reality Brief с evidence  
3. Feasibility — данные/инварианты не запрещают (или есть план)  
4. Options — выбран `D-*` для зависимых фич  
5. Acceptance — AC happy + ключевые fail  
6. Tracker write — только по явной просьбе пользователя  

## Параллелизм

Хорошо: code / db / process / integration на фазе 1.  
Плохо: писать Feature PRD параллельно «разведке на всякий случай».
