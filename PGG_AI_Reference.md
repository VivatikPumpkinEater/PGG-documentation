# PGG — AI Agent Reference (Подробная техническая документация)

Этот файл предназначен для ИИ-агента, который помогает пользователям работать с плагином **Procedural Generation Grid (PGG)** от FImpossible Creations. Содержит подробные описания портов, параметров, паттернов и рецептов.

---

## ЧАСТЬ 1: АРХИТЕКТУРА И ПОТОК ДАННЫХ

### 1.1 Три уровня системы

```
УРОВЕНЬ 1: Build Planner (Layout)
  → Определяет ГДЕ будут комнаты/коридоры/зоны
  → Работает с FieldPlanner'ами и CheckerField3D (сетками ячеек)
  → Ноды PR_ управляют формой, позицией, коллизиями полей

УРОВЕНЬ 2: Field Setup (Spawning Rules)  
  → Определяет ЧТО спавнится в каждой ячейке
  → Для каждой ячейки сетки запускает цепочку правил SR_
  → Правила: условия (когда спавнить) + трансформация (как спавнить)

УРОВЕНЬ 3: Mod Graph (Fine Control)
  → Тонкий контроль спавна на уровне отдельных ячеек
  → Ноды MR_ внутри нодового графа, запускаемого из SR_ModGraph
  → Доступ к текущему спавну, ячейке, сетке
```

### 1.2 Ключевые ScriptableObject-ассеты

| Ассет | Описание | Связь |
|-------|----------|-------|
| **BuildPlannerPreset** | Layout уровня | Содержит список FieldPlanner |
| **FieldPlanner** | Одна зона (комната, коридор) | Имеет ShapeGenerator + граф PR_ нод + FieldSetup |
| **FieldSetup** | Правила заполнения ячеек | Содержит ModificatorPack[] |
| **ModificatorsPack** | Пакет модификаторов | Содержит FieldModificator[] |
| **FieldModificator** | Один тип объекта (стены, полы) | Содержит Spawner[] с правилами SR_ |
| **PlannerFunctionNode** | Подграф-функция (без кода) | Переиспользуемый блок PR_ нод |

### 1.3 Компонент на сцене: BuildPlannerExecutor

Размещается на GameObject. Содержит:
- Ссылку на `BuildPlannerPreset`
- `PlannerPrepare[]` — для каждого FieldPlanner задаёт:
  - Какой FieldSetup использовать (через FieldSetupComposition)
  - Какой ShapeGenerator (InitShapes)
  - Количество дубликатов (экземпляров)

**Вызов `Generate()`** запускает весь пайплайн.

---

## ЧАСТЬ 2: СИСТЕМА ПОРТОВ

### 2.1 Типы портов

| Порт | Цвет | Тип данных | Конвертации |
|------|------|-----------|-------------|
| **IntPort** | — | `int` | ← float (усечение) |
| **FloatPort** | — | `float` | ← int |
| **BoolPort** | — | `bool` | — |
| **PGGVector3Port** | — | `Vector3` | ← Vector2, CellPort (позиция) |
| **PGGStringPort** | — | `string` | ← почти любой тип |
| **PGGUniversalPort** | фиолетовый | любой через `FieldVariable` | bool, int, float, Vector2/3, string, Color, Curve, Object |
| **PGGPlannerPort** | оранжевый | `FieldPlanner` / `CheckerField3D` | ← int (индекс поля), float, Vector |
| **PGGCellPort** | — | `FieldCell` + `CheckerField3D` | — |
| **PGGModCellPort** | — | ячейки мод-графа | — |
| **PGGSpawnPort** | — | `SpawnData` | — |
| **PGGTriggerPort** | — | сигнал выполнения | — |

### 2.2 Два типа исполнения нод

**Execution-ноды** (с trigger-коннекторами):
- Имеют входной и/или выходной коннекторы потока (стрелки сверху/снизу)
- Выполняются последовательно: PE_Start → нода1 → нода2 → ...
- Метод `Execute()` содержит логику
- Могут иметь несколько выходов (ветвление): Fail/Success, True/False, Iteration/Finish

**Read-ноды** (без trigger-коннекторов, `DrawInputConnector = false`):
- Не участвуют в цепочке выполнения
- Вычисляют значение при чтении порта (`OnStartReadingNode`)
- Примеры: Get Field Position, Get Bounds, Add, Collect All Fields

---

## ЧАСТЬ 3: BUILD PLANNER — ПОТОК ВЫПОЛНЕНИЯ

### 3.1 Фазы выполнения графа

```
1. First Procedures (основной граф)
   PE_Start → OutputConnection → первая нода → цепочка через OutputConnections
   
2. Sub Graphs (AfterEachInstance) — после каждого экземпляра
3. Sub Graphs (Default/Mid) — после всех первых процедур всех полей
4. Sub Graphs (PostProcedure) — перед post
5. Post Procedures (пост-граф)
   PE_Start (post) → OutputConnection → цепочка
```

### 3.2 Как ноды соединяются

**Trigger-связи** (стрелки сверху/снизу ноды):
- Определяют порядок выполнения
- PE_Start → Tight Placement → Push Out → Iterate Cells → ...
- Некоторые ноды имеют **несколько выходов**:
  - `Tight Placement`: выход 0 = Fail, выход 1 = Success
  - `Iterate Cells`: выход 0 = Finish, выход 1 = Iteration (тело цикла)
  - `Path Find Generate`: выход 0 = On Fail, выход 1 = On Found
  - `If A==B`: выход 0 = False, выход 1 = True

**Port-связи** (провода между полями нод):
- Передают данные: числа, векторы, планировщики, ячейки
- Вычисляются при чтении (ленивое вычисление)

### 3.3 Контекст выполнения

Каждая PR_ нода имеет доступ к:
- `CurrentExecutingPlanner` — текущий FieldPlanner
- `newResult` / `LatestResult` — результат (CheckerField3D с ячейками)
- `print` — PlanGenerationPrint (метаданные генерации)

---

## ЧАСТЬ 4: ПОДРОБНЫЕ ОПИСАНИЯ КЛЮЧЕВЫХ НОД

### 4.1 Build Planner — Размещение полей

#### Tight Placement
**Inspector:** «Tight Placement»  
**Назначение:** Прижать текущее поле вплотную к другому.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| AlignTo | PGGPlannerPort | Input | К какому полю прижимать |
| PushOut | FloatPort | Input | Расстояние отталкивания (по умолч. 1) |
| SidesSpace | FloatPort | Input | Зазор по бокам |
| ContactCell | PGGCellPort | Output | Ячейка контакта (текущего поля) |
| AlignedToCell | PGGCellPort | Output | Ячейка контакта (целевого поля) |

**Выходы:** 0 = Fail (не удалось разместить), 1 = Success  
**Параметры:** `FastCheck` (bool) — быстрая проверка, `ForceForwardAlign` (bool, в foldout)

---

#### Push Out Collision
**Inspector:** «Push Out Collision»  
**Назначение:** Вытолкнуть поле из коллизии с другими.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| CollisionWith | PGGPlannerPort | Input | С чем проверять коллизию |
| ToPush | PGGPlannerPort | Input | Какое поле толкать (по умолч. текущее) |

**Логика:** До 32 итераций пытается вытолкнуть поле. Если порт пуст — проверяет со всеми полями.

---

#### Align Self To
**Inspector:** «Align Self To»  
**Назначение:** Выровнять текущее поле относительно другого.

---

#### Push Until not Collides
**Inspector:** «Push Until not Collides»  
**Назначение:** Толкать поле в заданном направлении, пока не перестанет пересекаться.

---

#### Quick Align
**Inspector:** «Quick Align»  
**Назначение:** Быстрое выравнивание по альтернативному алгоритму.

---

### 4.2 Build Planner — Генерация форм

#### Rect Generate
**Inspector:** «Rect Generate»  
**Назначение:** Создать прямоугольную форму.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Width | IntPort | Input | Ширина |
| Height | IntPort | Input | Высота (Y) |
| Depth | IntPort | Input | Глубина |
| CenterOrigin | BoolPort | Input | Центрировать origin |
| RectShape | PGGPlannerPort | Output | Результирующая форма |

---

#### Line Generate
**Inspector:** «Line Generate»  
**Назначение:** Создать линию/коридор между двумя точками.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| PathStart | PGGVector3Port | Input | Начало линии |
| PathEnd | PGGVector3Port | Input | Конец линии |
| Radius | IntPort | Input | Радиус (толщина) |
| PathShape | PGGPlannerPort | Output | Результирующая форма |

**Параметры:** `TryStartCentered`, `RemoveFinishCell`, `NonDiag` (в foldout)

---

#### Path Find Generate
**Inspector:** «Path Find Generate»  
**Назначение:** Найти путь A* между двумя точками, обходя коллизии.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| StartOn | PGGPlannerPort | Input | Откуда начинать |
| SearchTowards | PGGPlannerPort | Input | Куда искать |
| CollideWith | PGGPlannerPort | Input | С чем коллизии (стены) |
| PathShape | PGGPlannerPort | Output | Найденный путь |
| From_StartCell | PGGCellPort | Output | Стартовая ячейка |
| Path_StartCell | PGGCellPort | Output | Начало пути |
| Path_EndCell | PGGCellPort | Output | Конец пути |
| Towards_EndCell | PGGCellPort | Output | Целевая ячейка |
| PathBase | PGGPlannerPort | Input | База пути (в foldout) |
| SingleStepPath | BoolPort | Output | Путь в 1 шаг? |

**Выходы:** 0 = On Fail, 1 = On Found  
**Параметры:** `PositionsMode`, `RemoveOverlappingCells`, `PathLengthZeroIsFail`, `PathfindSetup` (в отдельном окне)

---

### 4.3 Build Planner — Работа с ячейками

#### Iterate Cells
**Inspector:** «Iterate Cells»  
**Назначение:** Цикл по всем ячейкам поля.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| IterationCell | PGGCellPort | Output | Текущая ячейка итерации |
| CellsOf | PGGPlannerPort | Input | Чьи ячейки перебирать (в foldout) |
| BreakLoop | BoolPort | Input | Прервать цикл (в foldout) |

**Выходы:** 0 = Finish, 1 = Iteration  
**Параметры:** `RandomizeCellsOrder` (bool)

---

#### Add Cell Instruction
**Inspector:** «Add Cell Instruction»  
**Назначение:** Добавить инструкцию (doorway, ghost и т.д.) к ячейке.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Cell | PGGCellPort | Input | Целевая ячейка |
| ID | IntPort | Input | ID инструкции |
| DataString | PGGStringPort | Input | Строковые данные |
| Dir | PGGVector3Port | Input | Направление |
| Tags | PGGStringPort | Input | Теги |

**Параметры:** `Operation` (enum), `UseDirection`, `FlattenDiagonalDir`, `PreventDuplicates`

---

#### Get Cell Position
**Inspector:** «Get Cell Position»  
**Назначение:** Получить мировую/локальную позицию ячейки.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Cell | PGGCellPort | Input | Ячейка |
| Position | PGGVector3Port | Output | Позиция |

**Параметры:** `PositionSpace` — World / Local

---

### 4.4 Build Planner — Логика

#### If A==B : Execute A or B
**Inspector:** «If A==B : False/True» (знак зависит от LogicType)  
**Назначение:** Сравнить два значения и выполнить ветку.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| AValue | PGGUniversalPort | Input | Значение A |
| BValue | PGGUniversalPort | Input | Значение B |

**Выходы:** 0 = False, 1 = True  
**Параметры:** `LogicType` (==, >, >=, <, <=), `Negate` (инвертировать результат)

---

#### Iterator (Loop)
**Inspector:** «Iterator (Loop)»  
**Назначение:** Простой цикл for.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Iterations | IntPort | Input | Количество итераций |
| IterationIndex | IntPort | Output | Текущий индекс |
| Stop | BoolPort | Input | Прервать цикл |

**Выходы:** 0 = Finish, 1 = Iteration

---

### 4.5 Build Planner — Данные полей

#### Get Bounds
**Inspector:** «Get Bounds»  
**Назначение:** Получить Bounds поля. Read-нода.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Planner | PGGPlannerPort | Input | Источник |
| Center | PGGVector3Port | Output | Центр bounds |
| Size | PGGVector3Port | Output | Размер |
| Diagonal | FloatPort | Output | Диагональ |
| Width | FloatPort | Output | Ширина (X) |
| Height | FloatPort | Output | Высота (Y) |
| Depth | FloatPort | Output | Глубина (Z) |
| Min | PGGVector3Port | Output | Минимальная точка |
| Max | PGGVector3Port | Output | Максимальная точка |

---

#### Collect All Fields
**Inspector:** «Collect All Fields»  
**Назначение:** Собрать все поля Build Planner в список. Read-нода.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| OnlyTagged | PGGStringPort | Input | Фильтр по тегу |
| GetDuplicates | BoolPort | Input | Включить дубликаты |
| MultiplePlanners | PGGPlannerPort | Output | Собранные поля |

**Параметры:** `IgnoreThisField`, `GetSubFields` (в foldout)

---

#### Iterate Fields
**Inspector:** «Iterate Fields»  
**Назначение:** Цикл по полям.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| FieldsToIterate | PGGPlannerPort | Input | Какие поля перебирать |
| IterationField | PGGPlannerPort | Output | Текущее поле |
| IterationIndex | IntPort | Output | Текущий индекс |
| Stop | BoolPort | Input | Прервать цикл |

**Выходы:** 0 = Finish, 1 = Iteration  
**Параметры:** `GetDuplicates` (bool, «Get Instances»)

---

#### Is Colliding With
**Inspector:** «Is Colliding With»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| CollidingWith | PGGPlannerPort | Input | С чем проверять |
| IsColliding | BoolPort | Output | Результат |
| FirstColliderField | PGGPlannerPort | Input | Проверяемое поле (в foldout) |

---

#### Add
**Inspector:** «Add»  
**Назначение:** Сложение значений или объединение списков полей. Read-нода.

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| InValA | PGGUniversalPort | Input | Значение A |
| InValB | PGGUniversalPort | Input | Значение B |
| OutVal | PGGUniversalPort | Output | Результат (число/вектор/строка) |
| OutPlanners | PGGPlannerPort | Output | Список полей (если входы — поля) |

---

### 4.6 Mod Graph — Ключевые ноды

#### Set Spawn Position
**Inspector:** «Set Spawn Position»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Position | PGGVector3Port | Input | Позиция/оффсет |
| Spawn | PGGSpawnPort | Input | Спавн (по умолч. текущий) |

**Параметры:**
- `Operation`: Set / Offset / Subtract
- `Measure`: Units / Cells (в Cells масштабирует на размер ячейки)
- `OffsetSpace`: WorldSpace (spawn.Offset) / Local (spawn.DirectionalOffset)

---

#### Set Spawn Rotation
**Inspector:** «Set Spawn Rotation»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Degrees | input | Input | Углы в градусах |
| Spawn | PGGSpawnPort | Input | Спавн |

**Параметры:** `Operation`, `OffsetSpace` (WorldRotation / LocalRotation / TempRotation)

---

#### Get Cell At
**Inspector:** «Get Cell At»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Offset | PGGVector3Port | Input | Смещение |
| ResultCell | PGGModCellPort | Output | Найденная ячейка |
| OriginCell | input | Input | Базовая ячейка (в foldout, по умолч. текущая) |

**Параметры:** `GetAt` — ExactPosition / OffsetFromCurrentCell

---

#### Cell State
**Inspector:** «Cell State»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| CheckCell | input | Input | Проверяемая ячейка |
| OccupiedBy | string input | Input | Тег для проверки Occupied |
| IsTrue | BoolPort | Output | Результат |
| FoundSpawn | PGGSpawnPort | Output | Найденный спавн (при Occupied) |

**Параметры:** `CellMustBe` (Empty / InGrid / OutOfGrid / Occupied), `CheckMode`, `NegateResult`, `MultiCheck` (AllNeeded / AtLeastOne)

---

#### Break Spawner / Restore Allow Spawn
Нет портов. Только триггер-вход.
- **Break Spawner**: `CellAllow = false` — блокирует спавн
- **Restore Allow Spawn**: `CellAllow = true` — восстанавливает

---

#### Generate Spawn
**Inspector:** «Generate Spawn»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Generated | PGGSpawnPort | Output | Созданный спавн |

**Параметры:** `ApplyPrefab` — None / Reference (конкретный префаб) / GetFromModificator (по индексу)

---

#### Get Field Variable
**Inspector:** «Get Field Variable»

| Порт | Тип | Направление | Описание |
|------|-----|-------------|----------|
| Out | PGGUniversalPort | Output | Значение переменной |

**Параметры:** `VariableIdx` (выбор из списка), `VariablesSource` (ParentFieldSetup / ParentModPack)

---

## ЧАСТЬ 5: FIELD SETUP — ПРАВИЛА СПАВНА (SR_)

### 5.1 Порядок выполнения правил

```
Для каждого Spawner в каждом FieldModificator:
  Для каждого Repeat:
    Для каждой ячейки (порядок: ordered/reversed/random):
      1. Глобальные pre-правила → CheckRuleOn
      2. Локальные правила по порядку в списке → Refresh + CheckRuleOn
         (OnConditionsMet правила ПРОПУСКАЮТСЯ здесь)
      3. Глобальные post-правила → CheckRuleOn
      4. RunSpawnMod: оценка CellAllow/Negate/AND/OR → CellInfluence
      5. При успехе: OnConditionsMetAction для типов OnConditionsMet/Coded
      6. Добавление спавна в ячейку
```

**ВАЖНО:** Порядок правил в списке имеет значение! Правила обрабатываются сверху вниз.

### 5.2 Типы процедур (EProcedureType)

| Тип | Фаза | Описание |
|-----|------|----------|
| **Rule** | CheckRuleOn | Только условие (allow/deny). Задаёт CellAllow |
| **Procedure** | CheckRuleOn + CellInfluence | Условие + модификация спавна |
| **Event** | CellInfluence | Всегда выполняется (CellAllow = true после Refresh), меняет спавн |
| **OnConditionsMet** | OnConditionsMetAction | Только после успешной проверки ВСЕХ правил |
| **Coded** | CheckRuleOn + OnConditionsMetAction | Кастомная комбинация |

### 5.3 Ключевые SR_ ноды

#### Wall Placer
**Inspector:** «Wall Placer»  
**Тип:** Coded  
**Назначение:** Автоматическая расстановка стен по периметру сетки.

**Параметры:** `Module` (меш стены), `CornerMode` (90°), `CornerMode45`, `SpawnOn` (условие), `OccupiedTags`, `SpawnOnEachSide`, `AutoRemoveOverlaps`, `SetGhosts`, `YawOffset`, `DirectOffset`

**Логика:** Анализирует соседей каждой ячейки; для внешних граней ставит стену; углы заполняет Corner-мешами; удаляет пересечения через stigma.

---

#### Floor Placer
**Inspector:** «Floor Placer»  
**Тип:** Rule  
**Назначение:** Расстановка полов с учётом краёв.

**Параметры:** `Mode`, `AlignMode`, `YawOffset`, `DirectOffset`, `RotateBy90Only`  
В Advanced: `SpawnOnCheck`, `SpawnOnTag`, `CheckMode`, `OutsideCondition`, `OutsideOnTag`

---

#### Doorway Placer
**Inspector:** «Doorway Placer»  
**Тип:** Coded  
**Назначение:** Размещение дверного проёма.

**Параметры:** `replaceOnTag`, `CheckMode`, `Offset`, `OffsetMode`, `YawRotationOffset`, `RemoveDistance`, `DistanceSource`

**Логика:** Использует `restrictDirection` из инструкции ячейки (DoorHole command); находит ближайший спавн с тегом в радиусе RemoveDistance и удаляет его.

---

#### Check cell neightbours
**Inspector:** «Check cell neightbours»  
**Тип:** Rule  
**Назначение:** Условие по состоянию соседних ячеек.

**Параметры:** `CheckedCellsMustBe` (Empty/InGrid/OutOfGrid/Occupied), `NeightbourNeeds` (один/все/маска), `occupiedByTag`, `CheckMode`, `DirectCheck`, `quartRotor`, `spawnOn`, `EachRotor`, `CustomCellsCheck`

**Логика:** Проверяет соседей по маске/роторам. При успехе может скопировать позицию/ротацию/масштаб с найденного спавна.

---

#### Move-Rotate-Scale
**Inspector:** «Move-Rotate-Scale»  
**Тип:** Event  
**Назначение:** Базовая трансформация спавна.

**Параметры:**
- `EffectOn`: Position / Rotation / Scale
- `ApplyMode`: Override / Additive
- `Space`, `GetCoords` (взять базу с другого спавна по тегу)
- `PositionOffset`, `RotationOffset`, `ScaleMultiplier`
- `RandomizeOffset`, `MaxDegreesSteps`

---

#### Mod Node Graph
**Inspector:** «Mod Node Graph»  
**Назначение:** Запуск нодового Mod Graph внутри правил спавна.

**Параметры:** `ExternalModGraph` (файл графа), `CallDuring` (OnChecking / OnInfluence)

**Логика:** Устанавливает контекст (Graph_Mod, Graph_Spawner, Graph_SpawnData, Graph_Cell, Graph_Grid), затем выполняет граф от PE_Start.

**ВАЖНО:** При `OnChecking` граф выполняется в фазе CheckRuleOn и может задать CellAllow. При `OnInfluence` — в фазе CellInfluence и может модифицировать спавн.

---

## ЧАСТЬ 6: ИНСТРУКЦИИ ЯЧЕЕК (Cell Instructions / Commands)

### Что такое «command» / инструкция

Инструкция (`SpawnInstruction`) — это метаданные на ячейке, определяющие специальное поведение:
- **DoorHole** — дверной проём (запускает модификатор дверей)
- **PreRunModificator** / **PostRunModificator** — запуск модификатора до/после
- **PreventAllSpawn** / **PreventSpawnSelective** — запрет спавна
- **InjectStigma** / **InjectDataString** — инъекция данных
- **SetGhostCell** — ячейка-призрак (без спавна, но учитывается алгоритмами)

### Как добавлять инструкции

**В Build Planner:** нода `Add Cell Instruction` (PR_AddCellInstruction)  
**В FieldSetup:** `CellsInstructions` в настройках FieldSetup  
**В коде:** `cell.AddCellInstruction(SpawnInstruction)`

### SpawnInstructionGuide

Сериализуемый «гайд» для инструкции: позиция, поворот, ID, направление. Метод `GenerateGuide()` создаёт `SpawnInstruction` для FieldSetup.

---

## ЧАСТЬ 7: РЕЦЕПТЫ — КАК РЕШАТЬ ТИПОВЫЕ ЗАДАЧИ

### 7.1 Создать комнату и разместить рядом с другой

```
Build Planner граф:
  PE_Start
    → [Tight Placement]
        AlignTo ← (оранжевый порт от другого FieldPlanner)
        выход Success →
          → [следующая нода...]
        выход Fail →
          → [Discard Field] (отбросить, если не разместилось)
```

**FieldPlanner настройки:**
- ShapeGenerator: `SG_RandomSizeRectangle` (случайный размер)
- Instances: количество дубликатов (комнат одного типа)

---

### 7.2 Соединить две комнаты коридором

```
Build Planner граф (у FieldPlanner коридора):
  PE_Start
    → [Path Find Generate]
        StartOn ← (планировщик комнаты A)
        SearchTowards ← (планировщик комнаты B)
        CollideWith ← (все поля через Collect All Fields)
        выход On Found →
          → [Join Field-Shape Cells]
              JoinWith ← PathShape (от Path Find)
        выход On Fail →
          → [Discard Field]
```

---

### 7.3 Проверить коллизию и вытолкнуть

```
PE_Start
  → [Tight Placement] (Success →)
    → [Is Colliding With]
        CollidingWith ← (все поля)
        IsColliding → [If => Execute A or B]
          True → [Push Out Collision]
                    CollisionWith ← (все поля)
          False → [следующая нода]
```

---

### 7.4 Добавить двери между комнатами

**В Build Planner:**
```
[Iterate Cells] (по ячейкам контакта между полями)
  → [Add Cell Instruction]
      Cell ← IterationCell
      Operation: Add DoorHole instruction
      Dir ← направление к соседнему полю
```

**В FieldSetup:**
- Создать FieldModificator для дверей
- Добавить правило `Doorway Placer` (SR_DoorwayPlacer)
- Настроить `replaceOnTag` (тег стены, которую заменяет дверь)
- Настроить `RemoveDistance` (радиус удаления стены)
- В FieldSetup → CellsInstructions → привязать модификатор дверей к DoorHole

---

### 7.5 Стены + пол в FieldSetup

**Структура FieldSetup:**
```
ModificatorsPack "Структура"
  ├── FieldModificator "Стены"
  │     └── Spawner: префаб стены
  │           └── Rule: Wall Placer
  │                Module = префаб стены
  │                AutoRemoveOverlaps = true
  │
  ├── FieldModificator "Полы"
  │     └── Spawner: префаб пола
  │           └── Rule: Floor Placer
  │                Mode = Default
  │
  └── FieldModificator "Потолки" (опционально)
        └── Spawner: префаб потолка
              └── Rule: Floor Placer + Move-Rotate-Scale (поворот 180°)
```

---

### 7.6 Условный спавн по позиции

```
Spawner в FieldModificator:
  1. [Grid Position] (SR_CellPosition)
       Mode: Between
       Axis: Y
       Range: 0..2
  2. [Check cell neightbours] (SR_CellNeightbours)
       CheckedCellsMustBe: OutOfGrid
       NeightbourNeeds: AtLeastOne
  3. [Move-Rotate-Scale] (SR_MoveRotateScale)
       EffectOn: Position
       RandomizeOffset: true
```

---

### 7.7 Использование Mod Graph для сложной логики спавна

**В FieldModificator → Spawner → Rules:**
```
1. [Mod Node Graph] (SR_ModGraph)
     CallDuring: OnInfluence
     
     Внутри Mod Graph:
       PE_Start
         → [Cell State] (MR_GetCellStateOnGrid)
             CellMustBe: Occupied
             OccupiedBy: "wall"
             IsTrue → [If true:]
               → [Set Spawn Position] (MR_SetPosition)
                   Operation: Offset
                   Position ← [Get Cell Neighbour → Get Cell Position]
               → [Set Spawn Rotation] (MR_SetRotation)
                   Operation: Set
                   Degrees ← [Command Direction]
```

---

### 7.8 Генерация нескольких этажей

```
Build Planner:
  FieldPlanner "Этаж" (Instances: 3)
  
  First Procedures:
    PE_Start
      → [Get Field Position] → позиция текущего
      → [Get Index Of Instance] → индекс этажа
      → [Multiply] (индекс × высота_этажа)
      → [Set Field Position] (Y = результат умножения)
      → [Tight Placement] / [Align Self To]
```

---

## ЧАСТЬ 8: SHAPE GENERATORS

| Generator | Inspector | Описание |
|-----------|-----------|----------|
| SG_NoShape | No Shape | Пустая форма |
| SG_StaticSizeRectangle | Static Rectangle | Фиксированный W×Y×D |
| SG_RandomSizeRectangle | Random Rectangle | Случайные размеры в диапазонах |
| SG_RandomSizeRectangleWithInstructions | Random Rect + Instructions | + инструкции по ячейкам |
| SG_ManualRectangles | Manual Rectangles | Выбор из ручных наборов |
| SG_LineGenerate | Basic Line | Линия/коридор |
| SG_DividedRectangle | Complex/Divided | Зона, разбитая на комнаты |
| SG_ShatteredRectangle | Complex/Shattered | Нарезка на фрагменты |
| SG_CrossRoad | Complex/Cross Road | Сетка улиц |
| SG_PerlinNoiseGenerating | Custom/Perlin | Высота по Perlin Noise |
| SG_RandomTunnels | Complex/Random Tunnels | Ветвящиеся туннели |
| SG_RandomTunnelsLimited | Complex/Random Tunnels (Limited) | Туннели с лимитом |
| SG_PrefabToGrid | Prefab to Grid | Сетка из bounds префаба |

---

## ЧАСТЬ 9: ГЛОССАРИЙ

| Термин | Описание |
|--------|----------|
| **CheckerField3D** | Трёхмерная сетка ячеек. Основная структура данных PGG |
| **FieldPlanner** | Планировщик одной зоны. Имеет форму, граф, FieldSetup |
| **FieldSetup** | Набор правил спавна объектов по ячейкам |
| **ModificatorsPack** | Пакет модификаторов (группа FieldModificator) |
| **FieldModificator** | Один модификатор (напр. "Стены"). Содержит Spawner[] |
| **Spawner** | Спавнер: префаб + список правил SR_ |
| **SpawnData** | Данные конкретного спавна: префаб, позиция, ротация, масштаб, теги |
| **Cell** | Ячейка сетки. Имеет позицию, данные, инструкции |
| **CellInstruction** | Инструкция на ячейке (door, prevent spawn и т.д.) |
| **Ghost Cell** | Ячейка-призрак: учитывается алгоритмами, но ничего не спавнит |
| **Stigma** | Строковая метка на спавне для идентификации другими правилами |
| **CellData** | Строковые данные на ячейке для передачи между правилами |
| **Tag** | Тег спавнера (не Unity tag). Используется для поиска спавнов |
| **Command** | = Cell Instruction. Инструкция на ячейке |
| **restrictDirection** | Направление из инструкции двери, используемое SR_ нодами |
| **PlannerResult** | Результат выполнения графа: CheckerField3D + инструкции |
| **Composition** | FieldSetupComposition — оверлей FieldSetup (замена переменных/префабов) |
| **Injection** | Подмена переменной FieldSetup перед генерацией |
| **SubField** | Подполе FieldPlanner — доп. сетка без отдельного экземпляра |
