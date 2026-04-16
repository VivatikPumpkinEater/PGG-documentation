# Procedural Generation Grid (PGG) — Полная документация

**Версия:** 1.6.6.2.12 (Beta)  
**Автор:** FImpossible Creations — Filip Moeglich  
**Пространство имён:** `FIMSpace.Generating`, `FIMSpace.Graph`

---

## Содержание

1. [Обзор архитектуры](#1-architecture-overview)
2. [Ключевые компоненты (MonoBehaviour)](#2-key-components)
3. [ScriptableObject-ассеты](#3-scriptableobject-assets)
4. [Система нодового графа (FGraph)](#4-node-graph-system)
5. [Система портов и типы данных](#5-port-system)
6. [Build Planner — Ноды планировщика (PR_)](#6-build-planner-nodes-pr)
7. [Mod Graph — Ноды модификации (MR_)](#7-mod-graph-nodes-mr)
8. [Field Setup — Правила спавна (SR_)](#8-field-setup-spawn-rules-sr)
9. [Функциональные ноды (FN_)](#9-function-nodes-fn)
10. [Ноды выполнения (PE_)](#10-execution-nodes-pe)
11. [Blueprint-ноды (BPNode_)](#11-blueprint-nodes-bpnode)
12. [Shape Generators — Генераторы форм](#12-shape-generators)
13. [Поток работы: от пресета до объектов](#13-workflow)
14. [Tile Designer](#14-tile-designer)
15. [Object Stamper & Pipe Generator](#15-object-stamper--pipe-generator)

---

## 1. Architecture Overview
### Обзор архитектуры

PGG — система процедурной генерации на основе сеток (grid). Архитектура состоит из нескольких слоёв:

```
BuildPlannerPreset (layout уровня)
  └── FieldPlanner[] (отдельные зоны/комнаты)
        ├── ShapeGenerator (форма зоны)
        ├── Нодовый граф процедур (PR_ ноды)
        └── DefaultFieldSetup (правила спавна)
              └── ModificatorPack[]
                    └── FieldModificator[]
                          └── Spawner[]
                                └── SR_ правила + Mod Graph (MR_ ноды)
```

### Иерархия классов нод

```
ScriptableObject
  └── FGraph_NodeBase              — базовый нод графа (порты, связи, отрисовка)
        └── PGGPlanner_NodeBase    — нод PGG (категории, видимость)
              └── PlannerRuleBase  — исполняемый нод (Execute, порты планировщика)
                    ├── PGGPlanner_ExecutionNode  — нод входа/управления потоком
                    └── PlannerFunctionNode       — контейнер функций (подграф)
```

---

## 2. Key Components
### Ключевые компоненты

### BuildPlannerExecutor
**Путь:** `PGG/Planners Related/BuildPlannerExecutor.cs`  
Основной компонент на сцене. Ссылается на `BuildPlannerPreset`, задаёт `PlannerPrepare` (какой FieldSetup и начальная форма для каждого планировщика). Вызов `Generate()` запускает цепочку: выполнение графов → создание GridPainter'ов → спавн объектов.

### GridPainter
**Путь:** `PGG/Components/GridPainter.cs`  
Наследник `PGGGeneratorBase`. Хранит сетку `FGenGraph`, позволяет ручную покраску ячеек в редакторе или автоматическое заполнение из Build Planner. Вызывает `GenerateObjects()` для спавна по `FieldSetup`.

### PGGGeneratorBase
**Путь:** `PGG/Core/Helper Components/Bases/PGGGeneratorBase.cs`  
Базовый класс генераторов: сид, `GenerateObjects()`, управление сгенерированными объектами.

### FlexibleGenerator
Альтернатива GridPainter при `FlexibleGen == true` — пошаговая генерация в корутине для рантайма.

### ObjectStampEmitter
**Путь:** `Object Stamper/ObjectStampEmitter.cs`  
Штамповка префабов через `OStamperSet`. Поддерживает raycast, физическую симуляцию, комбинирование.

---

## 3. ScriptableObject Assets
### ScriptableObject-ассеты

### BuildPlannerPreset
**Путь:** `PGG/Planners Related/BuildPlannerPreset.cs`  
Layout уровня. Содержит:
- `BuildVariables` — переменные уровня
- `BuildLayers[0].FieldPlanners` — список `FieldPlanner`
- `RunProceduresAndGeneratePrint(seed)` — запуск всех графов

### FieldPlanner
**Путь:** `PGG/Planners Related/FieldPlanner.cs`  
Одна зона/комната в layout. Содержит:
- `ShapeGenerator` — алгоритм формы (SG_*)
- `DefaultFieldSetup` — правила спавна
- `FProcedures` / `FPostProcedures` — графы нод
- `Instances` — дубликаты (комнаты одного типа)
- `FieldType`: FieldPlanner / BuildField / InternalField / Prefab
- `LatestResult` / `LatestChecker` — результат генерации

### FieldSetup (Field Spawning Setup)
**Путь:** `PGG/Core/Scriptables/FieldSetup.cs`  
Правила заполнения ячеек объектами. Содержит:
- `CellSize` / `NonUniformCellSize`
- `ModificatorPacks` — пакеты модификаторов
- `CellsInstructions` — команды для ячеек (doorway, wall и т.д.)
- `Variables` — переменные
- `OnSpawnProcessors` — процессоры спавна

### FieldSetupComposition
Оверлей поверх `FieldSetup`: замена переменных, префабов, мод-паков.

### ModificatorsPack
Пакет `FieldModificator`, каждый из которых содержит спавнеры со списком правил `SR_`.

---

## 4. Node Graph System
### Система нодового графа

### FGraph_NodeBase
**Путь:** `Shared Tools/Node FGraph/Node Base/FGraph_NodeBase.cs`  
Абстрактный базовый класс всех нод. Хранит:
- `IndividualID` — уникальный ID в графе
- `inputPorts` / `outputPorts` — списки портов (`IFGraphPort`)
- `OutputConnections` / `InputConnections` — триггерные связи (`FGraph_TriggerNodeConnection`)
- `NodePosition` — позиция на графе
- `_EditorCollapse` — свёрнутый/развёрнутый вид

Ключевые виртуальные методы:
- `GetDisplayName()` — отображаемое имя
- `GetNodeColor()` — цвет ноды
- `GetNodeTooltipDescription` — подсказка
- `NodeSize` — размер
- `DrawInputConnector` / `DrawOutputConnector` — показ коннекторов потока
- `IsFoldable` — разворачиваемая нода
- `RefreshNodeParams()` — обновление параметров

### PGGPlanner_NodeBase
Добавляет:
- `Enabled` — вкл/выкл
- `CustomPath` — путь в меню создания нод
- `NodeType` (`EPlannerNodeType`) — категория
- `NodeVisibility` (`EPlannerNodeVisibility`) — видимость

#### EPlannerNodeType (категории нод)
| Значение | Описание |
|----------|----------|
| `Uncategorized` | Без категории |
| `Externals` | Внешние (функциональные порты) |
| `Math` | Математика |
| `ReadData` | Чтение данных |
| `WholeFieldPlacement` | Операции с полем/размещением |
| `CellsManipulation` | Манипуляции с ячейками |
| `Logic` | Логика и ветвление |
| `Debug` | Отладка |
| `Cosmetics` | Косметика (комментарии) |
| `ModGraphNode` | Ноды мод-графа |

### PlannerRuleBase
**Путь:** `PGG/Planners Related/Planner Logics/Utilities/PlannerRuleBase.cs`  
Исполняемая нода. Ключевые методы:
- `Execute(PlanGenerationPrint, PlannerResult)` — основная логика
- `PreGeneratePrepare()` / `OnCustomPrepare()` / `Prepare()` — подготовка
- `GetPlannerFromPort()` — получение планировщика из порта
- `GetCellFromInputPort()` — получение ячейки из порта
- `CallOtherExecution()` — вызов следующей ноды в цепочке

`DrawInputConnector == true` по умолчанию — участвует в trigger-графе.

---

## 5. Port System
### Система портов

### Базовый класс
`NodePortBase` — в `Shared Tools/Node FGraph/Node Elements/FGraph_Scr_Port.cs`

Порты объявляются в нодах через атрибут `[Port]`:
```csharp
[Port(EPortPinType.Input)] public PGGCellPort Cell;
[Port(EPortPinType.Output, EPortNameDisplay.Default, "Result")] public PGGVector3Port Result;
```

### Типы портов

| Тип порта | Тип данных | Описание |
|-----------|-----------|----------|
| **BoolPort** | `bool` | Логическое значение |
| **IntPort** | `int` | Целое число |
| **FloatPort** | `float` | Дробное число |
| **PGGVector3Port** | `Vector3` | Трёхмерный вектор |
| **PGGStringPort** | `string` | Строка (принимает почти все типы) |
| **PGGPlannerPort** | `FieldPlanner` / `CheckerField3D` | Планировщик или чекер (оранжевый порт). Может принимать int/float/Vector для индексации |
| **PGGUniversalPort** | любой через `FieldVariable` | Универсальный порт — bool, int, float, Vector2/3, string, Color, Curve, Object |
| **PGGCellPort** | `FieldCell` + `CheckerField3D` | Ячейка сетки с привязкой к чекеру |
| **PGGModCellPort** | ячейки мод-графа | Ячейки для системы mod graph |
| **PGGSpawnPort** | `SpawnData` | Данные спавна (префаб, позиция, ротация и т.д.) |
| **PGGTriggerPort** | сигнал | Управление потоком выполнения (не данные) |

### Поток выполнения
1. **Trigger connections** (`OutputConnections`/`InputConnections`) — основная цепочка выполнения. `PE_Start` → первая нода → следующая через `CallOtherExecution()`.
2. **PGGTriggerPort** — явный триггер через порт, вызывает `CallExecution()` на целевой ноде.
3. **Чтение данных** — `GetPortValue()` рекурсивно тянет значение по связям портов.

---

## 6. Build Planner Nodes (PR_)
### Ноды планировщика

**178 нод** в `PGG/Planners Related/Planner Coded Nodes/`

### 6.1 Math (35 нод)

| Нода | Описание |
|------|----------|
| **PR_Add** | Сложение значений. С полями — объединение в список |
| **PR_Subtract** | Вычитание. С полями — убирает поля из списка |
| **PR_Multiply** | Умножение. С полями — создаёт список полей |
| **PR_Divide** | Деление |
| **PR_Half** | Делит значение на 2 |
| **PR_ABS** | Модуль (абсолютное значение) |
| **PR_Invert** | Инверсия значения (-1 * v) |
| **PR_OneMinus** | 1 − значение |
| **PR_Clamp** | Ограничение значения диапазоном |
| **PR_Round** | Округление |
| **PR_RoundPosition** | Округление позиции до сетки |
| **PR_Lerp** | Линейная интерполяция |
| **PR_Normalize** | Нормализация вектора |
| **PR_Magnitude** | Длина вектора / расстояние |
| **PR_DotProduct** | Скалярное произведение / угол между векторами |
| **PR_Cross** | Векторное произведение (X↔Z) |
| **PR_Append** | Собрать X, Y, Z в Vector3 |
| **PR_Split** | Разобрать значение на компоненты |
| **PR_V3Get** | Получить X/Y/Z компоненту Vector3 |
| **PR_GetValueByAxis** | Получить компоненту по оси |
| **PR_DominantAxis** | Доминантная ось вектора |
| **PR_AngleDirection** | Угол направления |
| **PR_RotateDirection** | Поворот направления |
| **PR_DirectionToRotation** | Направление → эйлеры (Look Direction) |
| **PR_RotationToDirection** | Эйлеры → направление |
| **PR_WrapAngle** | Нормализация угла (0–360 или ±180) |
| **PR_GetRandom** | Случайное число в диапазоне |
| **PR_GenerateRandom** | Случайное число с входными параметрами |
| **PR_ChooseRandom** | Случайный выбор из нескольких входов |
| **PR_MathMinMax** | Выбрать большее / меньшее (Min/Max) |
| **PR_PeriodModulo** | Периодический modulo (%) |
| **PR_BoolTrigger** | Триггер boolean значения |
| **PR_ToBool** | Конвертация значения в bool |
| **PR_Value** | Вывод значения произвольного типа |
| **PR_Null** | Вывод null |

### 6.2 Cells — Работа с ячейками (30 нод)

| Нода | Тип | Описание |
|------|-----|----------|
| **PR_IterateCells** | CellsManipulation | Цикл по всем ячейкам поля |
| **PR_IterateCellsInLine** | CellsManipulation | Итерация ячеек по линии |
| **PR_IterateCellOutsideDirections** | CellsManipulation | Итерация по внешним направлениям ячейки |
| **PR_GetCellPosition** | ReadData | Получить мировую позицию ячейки |
| **PR_GetCell** | ReadData | Получить ячейку по мировой позиции |
| **PR_GetCellByIndex** | ReadData | Получить ячейку по индексу |
| **PR_GetNearestCell** | ReadData | Ближайшая ячейка к позиции |
| **PR_GetNearestCellIn** | ReadData | Ближайшая ячейка в наборе |
| **PR_GetRadomCellIn** | ReadData | Случайная ячейка из набора |
| **PR_GetRadomCellWithNoInstruction** | ReadData | Случайная ячейка без инструкций |
| **PR_GetMostMiddleCell** | CellsManipulation | Самая центральная ячейка |
| **PR_GetMostCellsDirection** | ReadData | Направление с наибольшим числом ячеек |
| **PR_SelectCellsOnEdge** | CellsManipulation | Выбрать ячейки на краю поля |
| **PR_GetCollidingCells** | ReadData | Ячейки, пересекающиеся с другим полем |
| **PR_IsAnyCellInPosition** | ReadData | Есть ли ячейка в позиции? |
| **PR_DetectCellIn** | ReadData | Детектировать ячейку в позиции |
| **PR_AreCellsDiagonalToEach** | ReadData | Проверка диагональности ячеек |
| **PR_CheckContactInDirection** | CellsManipulation | Контакт с полем в направлении |
| **PR_BoundsSweepCollisionInDirection** | ReadData | Sweep-коллизия bounds в направлении |
| **PR_RoundAccordingly** | CellsManipulation | Выровнять позицию под сетку другого поля |
| **PR_SetCellParameter** | ReadData | Установить/добавить параметр ячейки |
| **PR_GetCellParameters** | ReadData | Прочитать параметры ячейки |
| **PR_GetCellInstructionParams** | ReadData | Параметры инструкции ячейки |
| **PR_AddCellInstruction** | ReadData | Добавить инструкцию ячейке |
| **PR_CopyOtherCellParams** | ReadData | Копировать инструкции/данные из другой ячейки |
| **PR_CellContainsDataString** | ReadData | Проверить наличие строки в данных ячейки |
| **PR_IsCellDataInRange** | ReadData | Проверка диапазона данных ячейки |
| **PR_DirectionFromTo** | ReadData | Направление от/к позиции или ячейке |
| **PR_CellDebug** | ReadData | Отладочная информация ячейки |

### 6.3 Field Planner — Операции с полем (67 нод)

#### Доступ к данным поля (ReadData)

| Нода | Описание |
|------|----------|
| **PR_GetFieldGridBoundaries** | Границы сетки поля |
| **PR_CellsCount** | Количество ячеек |
| **PR_GetFieldBounds** | Bounds поля (поддерживает несколько полей → один Bounds) |
| **PR_GetBoundsParameter** | Параметр bounds (размер, центр и т.д.) |
| **PR_GetFieldParameter** | Параметры планировщика |
| **PR_GetCellInstructionCount** | Количество инструкций ячейки |
| **PR_GetCellInstruction** | Получить конкретную инструкцию ячейки |
| **PR_GetFieldSelector** | Получить FieldPlanner из селектора |
| **PR_FieldsCount** | Общее число полей |
| **PR_FieldInstancesCount** | Число экземпляров планировщика |
| **PR_SubFieldsCount** | Число подполей |
| **PR_GetSubField** | Получить подполе по индексу |
| **PR_GetSubFields** | Получить все подполя |
| **PR_GetFieldInstance** | Экземпляр планировщика по ID |
| **PR_GetFieldFromInt** | Планировщик по числовому индексу |
| **PR_GetGlobalIndexOfInstance** | Глобальный индекс экземпляра |
| **PR_GetPlannerDuplicate** | Дубликат планировщика |
| **PR_GetFieldDuplicates** | Все дубликаты |
| **PR_GetCellSize** | Размер ячейки сетки |
| **PR_ScaleWithFieldCellSize** | Масштабирование с учётом размера ячейки |

#### Поиск соседей и других полей

| Нода | Описание |
|------|----------|
| **PR_GetNeightbourField** | Ближайшее смежное поле |
| **PR_ChooseNeightbourFields** | Выбрать несколько смежных полей |
| **PR_GetNearestFieldPlanner** | Ближайший планировщик (с условием) |
| **PR_GetFarthestFieldPlanner** | Самый дальний планировщик |
| **PR_GetRadomFieldPlanner** | Случайный планировщик |

#### Проверки полей

| Нода | Описание |
|------|----------|
| **PR_IsFieldCollidingWith** | Пересекается ли поле с другим |
| **PR_IsFullyContainedBy** | Полностью ли поле содержится внутри другого |
| **PR_IsFieldAligningWith** | Выровнено ли поле с другим |
| **PR_CountAlignmentsWith** | Число выравниваний с другим полем |
| **PR_CheckBoundsCollisionBetween** | Коллизия bounds между полями |

#### Модификация формы

| Нода | Описание |
|------|----------|
| **PR_JoinShapeCells** | Объединить ячейки формы |
| **PR_RemoveFieldCells** | Удалить ячейки из поля |
| **PR_RemoveAllFieldCells** | Удалить все ячейки |
| **PR_ReplaceFieldCells** | Заменить ячейки |
| **PR_ShapeSubtract** | Вычесть одну форму из другой |
| **PR_ShapeUnion** | Объединение форм |
| **PR_SplitField** | Разделить поле |
| **PR_YToFields** | Y-координаты в отдельные поля |
| **PR_GetDisconnectedChunks** | Найти изолированные группы ячеек |
| **PR_RectangleDivide** | Разделить прямоугольную область |
| **PR_GenerateEmptyShape** | Пустой контейнер формы |
| **PR_AddSubField** | Добавить подполе |
| **PR_AssignSubFieldParameter** | Назначить параметр подполю |
| **PR_RemoveTooFarCells** | Удалить слишком далёкие ячейки |
| **PR_DiscardField** | Отбросить поле целиком |

#### Трансформация поля (WholeFieldPlacement)

| Нода | Описание |
|------|----------|
| **PR_GetFieldPosition** | Получить позицию поля |
| **PR_SetFieldPosition** | Установить позицию |
| **PR_RoundFieldPosition** | Округлить позицию поля |
| **PR_GetFieldRotation** | Получить ротацию поля |
| **PR_SetFieldRotation** | Установить ротацию |
| **PR_SetFieldOrigin** | Установить origin поля |
| **PR_CenterFieldOrigin** | Центрировать origin |

#### Размещение и коллизии

| Нода | Описание |
|------|----------|
| **PR_TightPlacement** | Плотное размещение (прижать к другому полю) |
| **PR_CustomTightPlacement** | Кастомное плотное размещение |
| **PR_PushOutOfCollision** | Вытолкнуть из коллизии |
| **PR_PushOutAway** | Вытолкнуть подальше |
| **PR_PushInDirUntilNotCollides** | Толкать в направлении пока не перестанет пересекаться |
| **PR_PushInDirUntilFullyContained** | Толкать пока полностью не войдёт |
| **PR_PushInDirToAlign** | Толкать для выравнивания |
| **PR_BoundsSeparatePushOut** | Разделить пересекающиеся bounds |
| **PR_AlignTo** | Выровнять к другому полю |
| **PR_QuickAlign** | Быстрое выравнивание (альтернативный алгоритм) |

#### Переменные

| Нода | Описание |
|------|----------|
| **PR_GetLocalVariable** | Прочитать локальную переменную |
| **PR_SetLocalVariable** | Установить локальную переменную |
| **PR_SetLocalVariableAlloc** | Установить переменную с сохранением между итерациями |
| **PR_GetInternalValueVariable** | Прочитать внутреннюю переменную экземпляра |
| **PR_SetInternalValueVariable** | Записать внутреннюю переменную |
| **PR_GetFieldPlannerVariable** | Прочитать переменную планировщика |

### 6.4 Logic — Логика и ветвление (13 нод)

| Нода | Описание |
|------|----------|
| **PR_EqualIfSwitch** | If compare ⇒ Output Value (сравнение → значение) |
| **PR_EqualIfExecution** | If compare ⇒ Execute A/B (сравнение → ветка выполнения) |
| **PR_EqualIfSwitchExecution** | If ⇒ Execute A/B |
| **PR_BoolIfSwitch** | If true/false ⇒ Output A or B |
| **PR_TrueFalseSwitch** | If compare ⇒ true/false |
| **PR_BoolAccumulate** | Накопление условий → bool |
| **PR_ValueBetweenSwitch** | Если значение в диапазоне |
| **PR_5050Execution** | 50% вероятность → Execute A или B |
| **PR_IsNull** | Проверка на null |
| **PR_IteratorLoop** | Цикл-итератор |
| **PR_RotorLoop** | Перебор поворотов (0°, 90°, 180°, 270°) |
| **PR_IterateList** | Цикл по списку object |

### 6.5 Build Setup (11 нод)

| Нода | Описание |
|------|----------|
| **PR_CollectFields** | Собрать все поля в список |
| **PR_ChooseFields** | Выбрать поля по условию |
| **PR_ChooseOneField** | Найти одно поле по условию (наибольшее, ближайшее и т.д.) |
| **PR_IterateFields** | Цикл по полям |
| **PR_GetIterationIndex** | Текущий индекс итерации |
| **PR_GetBuildAreaBounds** | Bounds области генерации |
| **PR_GetBuildVariable** | Переменная Build Planner |
| **PR_ExpandBoundsSize** | Расширить bounds |
| **PR_GenerateBounds** | Сгенерировать bounds |
| **PR_ScheduleFieldInjection** | Запланировать инъекцию переменной в FieldSetup |
| **PR_CallGraph** | Вызвать другой подграф |

### 6.6 Generating — Генерация форм (10 нод)

| Нода | Описание |
|------|----------|
| **PR_RectGenerate** | Сгенерировать прямоугольник заданных размеров |
| **PR_LineGenerate** | Сгенерировать линию/коридор |
| **PR_PathFindGenerate** | Поиск пути A* между точками |
| **PR_FindPathTowards** | Pathfind к целевой позиции |
| **PR_BoundsToCells** | Конвертировать bounds в ячейки |
| **PR_AddCellToField** | Добавить ячейку к полю |
| **PR_GetInlineShape** | Получить внутренний контур формы |
| **PR_GetOutlineShape** | Получить внешний контур формы |
| **PR_GetFattenShape** | Утолстить тонкие пути |
| **PR_GetScaleConvertedShape** | Конвертировать форму с другим масштабом |

### 6.7 Utilities & Debug (8 нод)

| Нода | Описание |
|------|----------|
| **PR_Comment** | Комментарий на графе |
| **PR_Group** | Группировка нод |
| **PR_ConsoleLog** | Debug.Log |
| **PR_DebugDrawFieldHighlight** | Визуализация поля в Scene View |
| **PR_DebugDrawPosition** | Визуализация позиции |
| **PR_ToString** | Конвертация в строку |
| **PR_OneCallFilter** | Фильтр однократного вызова |
| **PR_Rewire** | Переподключение execution-потока |

### 6.8 Specific Solutions (2 ноды)

| Нода | Описание |
|------|----------|
| **PR_PathFind_DeloneConnections** | Связи комнат по Delaunay (Bowyer–Watson) |
| **PR_ApplyPerlinNoiseOffset** | Смещение высоты по Perlin Noise |

### 6.9 Community (2 ноды)

| Нода | Описание |
|------|----------|
| **PR_RemoveCellsUnder** | Удалить ячейки «под» другими |
| **PR_PathFindGenerateLowerCost** | Pathfind с пониженной стоимостью на указанных чекерах |

---

## 7. Mod Graph Nodes (MR_)
### Ноды модификации

**40 нод** в `PGG/Mod Graph/`  
Используются в Mod Node Graph внутри FieldSetup для управления спавном на уровне отдельных ячеек.

### 7.1 Управление спавном

| Нода | Описание |
|------|----------|
| **MR_AllowSpawn** | Разрешить спавн (восстановить после Break) |
| **MR_BreakSpawner** | Запретить спавн |
| **MR_DisableSpawningMainPrefab** | Отключить спавн основного префаба |
| **MR_GenerateSpawn** | Создать новые данные спавна |
| **MR_ApplyPrefabToSpawn** | Назначить префаб для спавна |
| **MR_AddExtraSpawn** | Добавить дополнительный спавн в очередь ячейки |
| **MR_CopySpawn** | Копировать данные спавна |
| **MR_RemoveSpawn** | Удалить спавн из очереди |
| **MR_AddSpawnStigma** | Добавить строковую стигму спавну |
| **MR_AddCellData** | Добавить Cell Data строку ячейке |

### 7.2 Чтение позиции / ротации / масштаба

| Нода | Описание |
|------|----------|
| **MR_GetPosition** | Позиция спавна |
| **MR_GetRotation** | Ротация спавна |
| **MR_GetScale** | Масштаб спавна |
| **MR_SetPosition** | Установить/сместить позицию |
| **MR_SetRotation** | Установить/сместить ротацию |
| **MR_SetScale** | Установить масштаб |
| **MR_ClearOffsets** | Сбросить все оффсеты |

### 7.3 Работа с ячейками

| Нода | Описание |
|------|----------|
| **MR_GetCellAt** | Ячейка по смещению от текущей |
| **MR_GetCellPosition** | Мировая позиция ячейки сетки |
| **MR_GetCellStateOnGrid** | Состояние ячейки (в сетке / вне / занята тегом) |
| **MR_GetCellsAround** | Ячейки в квадрате вокруг |
| **MR_GetNeighbourCell** | Соседняя ячейка (с поворачиваемым смещением) |
| **MR_GetFillConnectedCells** | Все достижимые ячейки (fill, без диагоналей) |
| **MR_GetNearestCellWith** | Ближайшая ячейка с условием |
| **MR_GetOtherCellInDistance** | Расстояние до ячеек по пути |
| **MR_IterateCells** | Итерация по списку ячеек |
| **MR_IterateCellInstructions** | Итерация по инструкциям ячейки |
| **MR_LineCheckCells** | Проверка ячеек по линии в 4 направлениях |

### 7.4 Данные спавна

| Нода | Описание |
|------|----------|
| **MR_GetSpawnFromCell** | Первый/все спавны ячейки с параметрами |
| **MR_GetSpawnsInCell** | Все спавны в ячейке |
| **MR_GetSpawnPrefab** | Ссылка на спавнимый префаб |
| **MR_GetPrefabBounds** | Bounds префаба |
| **MR_SpawnIsTagged** | Проверка тега спавна |
| **MR_IterateSpawns** | Итерация по спавнам ячейки |

### 7.5 Другое

| Нода | Описание |
|------|----------|
| **MR_GetFieldVariable** | Прочитать переменную FieldSetup / ModPack |
| **MR_GetCommandDirection** | Направление команды ячейки |
| **MR_GetGridCellSize** | Размер ячейки сетки |
| **MR_GridSize** | Размеры сетки |
| **MR_TileDesigner** | Сгенерировать объект через Tile Designer |

---

## 8. Field Setup Spawn Rules (SR_)
### Правила спавна

**~85 правил** в `PGG/Rules Logics/`  
Используются в спавнерах FieldModificator для контроля процесса генерации объектов по ячейкам.

### Базовый класс: SpawnRuleBase
**Путь:** `PGG/Core/Scriptables/SpawnRuleBase.cs`  
Ключевые концепции:
- `EProcedureType`: Procedure, Rule, Event, OnConditionsMet, Coded, Output
- `ERuleLogic`: AND_Required, OR
- Жизненный цикл: `Refresh` → `ResetRule` → `CheckRuleOn` → `CellInfluence` → `OnConditionsMetAction`
- Флаги: `Negate`, `Global`, `Enabled`

### 8.1 Cells — Работа с ячейками

| Правило | Описание |
|---------|----------|
| **SR_CellOperation** | Операции над выбранной ячейкой |
| **SR_AnalyzeCell** | Проверка состояния ячейки (allow/deny) |
| **SR_AddCellDataString** | Записать cell data при условиях |
| **SR_CellSpawnsCount** | Условие по числу спавнов в ячейке |
| **SR_HideCell** | Скрытие ячейки |
| **SR_PreventModsSpawns** | Блокировка спавнов других модификаторов |
| **SR_PreventSpawns** | Блокировка следующих спавнеров по тегам |
| **SR_RemoveSpawn** | Удаление спавна |
| **SR_RemoveSpawnsTool** | Удаление спавнов по наборам условий |
| **SR_RemoveInPosition** | Удаление спавнов по расстоянию |
| **SR_RemoveInDirection** | Удаление по направлению (Legacy) |

### 8.2 Placement — Условия размещения

| Правило | Описание |
|---------|----------|
| **SR_CellNeightbours** | Гибкая проверка соседних ячеек (3D-ротор) |
| **SR_MultiCheckCellNeightbours** | Множественные условия по соседям |
| **SR_CellPosition** | Условие по позиции ячейки на сетке |
| **SR_IfWorldPosition** | Условие по мировой позиции |
| **SR_IfRotated** | Допуск по диапазону поворота ячейки |
| **SR_IfCellContainsTag** | Теги/стигма/data в ячейке |
| **SR_OnGridCorner** | Углы сетки |
| **SR_AllowEveryFew** | Каждые N ячеек |
| **SR_CheckCellsInLine** | Проверка ячеек по линии |
| **SR_FreeSpace** | Проверка свободного места (Alpha) |
| **SR_HelperPivotCorrection** | Коррекция координат для выравнивания |
| **SR_SimulatePhysics** | Физическая симуляция после спавна |

### 8.3 Transforming — Трансформация

| Правило | Описание |
|---------|----------|
| **SR_MoveRotateScale** | Базовый оффсет позиции/ротации/масштаба |
| **SR_GetPosRotScale** | Получить PRS от другого спавна |
| **SR_PrefabOffset** | Оффсет из координат префаба |
| **SR_ConditionalPushPosition** | Push с разными значениями по углу |
| **SR_PerlinNoiseRotationOrScale** | Perlin Noise для ротации/масштаба |

#### Legacy Transforming

| Правило | Описание |
|---------|----------|
| **SR_Scale** | Масштабирование |
| **SR_WorldOffset** | Смещение позиции (без ротации) |
| **SR_DirectOffset** | Прямое смещение |
| **SR_PushPosition** | Аддитивный оффсет |
| **SR_ScaleRange** | Случайный масштаб в диапазоне |
| **SR_GetCoords** | Координаты от спавна по тегу |
| **SR_GetRestrictedRotation** | Ротация из command ячейки |
| **SR_GetCommandRotation** | Ротация из command |
| **SR_ChangeRotationPivot** | Поворот вокруг другого pivot |

### 8.4 Modelling — Генерация объектов

| Правило | Описание |
|---------|----------|
| **SR_TileDesigner** | Генерация через Tile Designer |
| **SR_RandomMesh** | Mesh Renderer со случайным мешем |
| **SR_CableGenerator** | Процедурный кабельный меш |
| **SR_ReplaceFilterMesh** | Замена mesh у MeshFilter |
| **SR_ReplaceWithRandomPrefab** | Замена префаба на случайный |
| **SR_ReplacePrefab** | Замена префаба + переменные |
| **SR_SetMaterial** | Назначение материала |
| **SR_RandomMaterial** | Случайный материал из списка |
| **SR_SetGameObjectLayer** | Установка layer у GameObject |
| **SR_AcquireSpawn** | Временный спавн для «пустого» слота |
| **SR_CombineMesh** | Отложенное объединение мешей |
| **SR_VolumeIndicator** | Маркер объёма |

#### Roof Generator

| Правило | Описание |
|---------|----------|
| **SR_RoofPlaneGenerator** | Генерация плоскости крыши |
| **SR_RoofSideGenerator** | Боковые стены крыши |
| **SR_RoofGeneratorPlacer** | Размещение элементов крыши |
| **SR_RoofGeneratorEdgesPlacer** | Рёбра/гребень крыши |

### 8.5 Quick Solutions — Быстрые решения

| Правило | Описание |
|---------|----------|
| **SR_WallPlacer** | Размещение стен с выравниванием |
| **SR_FloorPlacer** | Размещение полов |
| **SR_DoorwayPlacer** | Дверные проёмы из guide |
| **SR_DoubleDoorsHelper** | Двойные двери → один проём |
| **SR_CornerDoorwayCheck** | Команда vs поворот угла |
| **SR_StairsHelper** | Лестницы |
| **SR_EdgesPlacer** | Плитки по краям |
| **SR_DirectionalRemove** | Удаление по смещённой позиции |
| **SR_CallRulesOf** | Вызов правил другого спавнера |
| **SR_ModGraph** | Логика через Mod Node Graph |
| **SR_SubSpawner** | Доп. спавн (deprecated) |
| **SR_CablesSpawner** | Кабели (deprecated) |

### 8.6 Collision — Коллизии

| Правило | Описание |
|---------|----------|
| **SR_SpawnUntilCollides** | Спавн до коллизии |
| **SR_CheckWorldCollision** | Проверка коллизий со сценой |
| **SR_BoundCollisionOffset** | Оффсет по bounds + коллизии (Legacy) |
| **SR_OffsetFromBounds** | Оффсет от bounds другого объекта (Legacy) |

### 8.7 PostEvents — Пост-обработка

| Правило | Описание |
|---------|----------|
| **SR_FlattenTerrain** | Выравнивание Terrain |
| **SR_CutTerrainHole** | Дыра в Terrain |
| **SR_RemoveTerrainTrees** | Удаление деревьев Terrain |
| **SR_RemoveTerrainDetail** | Удаление деталей Terrain |

### 8.8 Operations — Операции

| Правило | Описание |
|---------|----------|
| **SR_StackSpawner** | Стек префабов (stamper) |
| **SR_PipeSpawner** | Pipe Generator без отдельного пресета |
| **SR_DuplicateSpawns** | Дублирование спавнов |

### 8.9 Count — Ограничения

| Правило | Описание |
|---------|----------|
| **SR_SpawningPropability** | Вероятность спавна |
| **SR_LimitSpawnCount** | Лимит числа спавнов |

### 8.10 FieldAndGrid — Поле и сетка

| Правило | Описание |
|---------|----------|
| **SR_GridSpecifics** | Условия по специфике сетки |
| **SR_IsSameFieldSetup** | Проверка Field Setup |
| **SR_OffsetCenter** | Смещение к центру сетки |
| **SR_CompareVariable** | Сравнение переменной |
| **SR_ShiftTowards** | Сдвиг к цели |

### 8.11 Other

| Правило | Описание |
|---------|----------|
| **SR_Separator** | Визуальный разделитель |
| **SR_Comment** | Комментарий/заметка |
| **SR_DebugLog** | Debug.Log при условиях |

---

## 9. Function Nodes (FN_)
### Функциональные ноды

Используются внутри `PlannerFunctionNode` — подграфы без компиляции.

| Нода | Тип | Описание |
|------|-----|----------|
| **FN_Input** | Externals | Входной порт функции |
| **FN_Output** | Externals | Выходной порт функции |
| **FN_Parameter** | Externals | Параметр функции (настраиваемое поле) |
| **FN_ExecuteOnRead** | Externals | Выполнить при чтении порта |
| **FN_ReadOnExecute** | Externals | Вычислить при execution (без входного trigger) |

### PlannerFunctionNode
**Путь:** `PGG/Planners Related/Planner Logics/PlannerFunctionNode.cs`  
Контейнер-функция, создаётся как ассет через:
`Create → FImpossible Creations → Procedural Generation → Create Build Planner Function Node`

Содержит вложенный граф нод. Позволяет создавать переиспользуемые блоки логики без написания кода.

---

## 10. Execution Nodes (PE_)
### Ноды выполнения

| Нода | Описание |
|------|----------|
| **PE_Start** | Начальная нода — точка входа в граф процедур |

`PE_Continue` закомментирован и не активен.

---

## 11. Blueprint Nodes (BPNode_)
### Blueprint-ноды

Кастомные ноды для специальных задач:

| Нода | Описание |
|------|----------|
| **BPNode_RectShape** | Генерация прямоугольной формы (шаблон для кастомных нод) |
| **BPNode_WindowsPlacer** | Размещение команд (окон) по периметру поля |

---

## 12. Shape Generators
### Генераторы форм

**Базовый класс:** `ShapeGeneratorBase` (`PGG/Planners Related/Planner Logics/Utilities/ShapeGeneratorBase.cs`)  
Метод: `GetChecker(FieldPlanner) → CheckerField3D`

| Генератор | Описание |
|-----------|----------|
| **SG_NoShape** | Пустая форма |
| **SG_StaticSizeRectangle** | Прямоугольник фиксированного размера W×Y×D |
| **SG_RandomSizeRectangle** | Прямоугольник со случайными размерами в диапазонах |
| **SG_RandomSizeRectangleWithInstructions** | Случайный прямоугольник + инструкции по ячейкам |
| **SG_ManualRectangles** | Один из заданных вручную наборов прямоугольников |
| **SG_LineGenerate** | Линия/коридор между точками |
| **SG_DividedRectangle** | Зона, разбитая на комнаты-кластеры |
| **SG_ShatteredRectangle** | Прямоугольник, нарезанный на фрагменты |
| **SG_CrossRoad** | Сетка улиц/кварталов |
| **SG_PerlinNoiseGenerating** | Y-высота ячеек по Perlin Noise |
| **SG_RandomTunnels** | Ветвящиеся туннели |
| **SG_RandomTunnelsLimited** | Туннели с ограничением размеров |
| **SG_PrefabToGrid** | Сетка из bounds префаба |

---

## 13. Workflow
### Поток работы: от пресета до объектов

```
1. Создать BuildPlannerPreset
   └── Добавить FieldPlanner'ы (комнаты, коридоры, этажи)
         ├── Выбрать ShapeGenerator (форму)
         ├── Назначить FieldSetup (правила спавна)
         └── Настроить граф процедур (PR_ ноды)

2. На сцене создать BuildPlannerExecutor
   └── Назначить BuildPlannerPreset
   └── Настроить PlannerPrepare (FieldSetupComposition на каждый планировщик)

3. Вызвать Generate()
   ├── Копия пресета, инициализация
   ├── Для каждого планировщика:
   │     ├── ShapeGenerator → CheckerField3D (начальная форма)
   │     └── Выполнение графов процедур (PE_Start → PR_* цепочка)
   │           └── Размещение, коллизии, модификация формы, pathfind
   │
   ├── RunGeneratePainters
   │     └── Для каждого результата:
   │           ├── Создать GridPainter на дочернем объекте
   │           ├── Скопировать сетку из LatestResult
   │           └── GenerateObjects() по FieldSetup:
   │                 └── Для каждой ячейки:
   │                       └── ModificatorPack → FieldModificator → Spawner
   │                             ├── SR_ правила (условия, трансформация)
   │                             └── Mod Graph (MR_ ноды)
   │                                   └── Инстанс префаба на сцене
   │
   └── Или для Prefab-типа: InstantiatePrefabField (просто инстанс)
```

### Жизненный цикл SR_ правила
```
PreGenerateResetRule → Refresh → ResetRule → CheckRuleOn(cell)
  ├── Проверка условий (AND/OR, Negate)
  ├── CellInfluence — влияние на ячейку
  └── OnConditionsMetAction → RunSpawnMod → EGRuleResult
```

---

## 14. Tile Designer

**Путь:** `Plugins - Level Design/Tile Designer/`

Инструмент для генерации архитектурных мешей (стены, крыши, столбы) без 3D-редактора. Поддерживает:
- Extrude (выдавливание)
- Loft (по кривой высоты)
- Sweep (арки, трубы, ветки)
- Примитивы (куб с фаской, сфера, цилиндр)
- Вырезание полигонов
- UV-контроль, нормали, комбинирование

Используется через:
- `SR_TileDesigner` (правило спавна)
- `MR_TileDesigner` (нода мод-графа)
- Отдельное окно Tile Designer Window

---

## 15. Object Stamper & Pipe Generator

### Object Stamper
**Путь:** `Plugins - Level Design/Object Stamper/`

Штамповка случайных объектов в физическом пространстве. Компоненты:
- `ObjectStampEmitter` — эмиттер штампов
- `MultiStampEmitter` — несколько штампов
- `OStamperSet` — пресет штамповки
- Поддержка raycast, физической симуляции, overlap-проверки

### Pipe Generator
**Путь:** `Plugins - Level Design/Pipe Generator/`

Генерация труб/кабелей по точкам. Используется через `SR_PipeSpawner` или отдельный компонент.

---

## Приложение: Структура папок

```
FImpossible Creations/
├── Plugins - Level Design/
│   ├── PGG/                          — Основной пакет
│   │   ├── Core/                     — Ядро (FieldSetup, SpawnRuleBase, CheckerField3D)
│   │   ├── Components/               — MonoBehaviour (GridPainter и т.д.)
│   │   ├── Planners Related/         — Build Planner (FieldPlanner, BuildPlannerPreset)
│   │   │   ├── Planner Coded Nodes/  — PR_ ноды (178 файлов)
│   │   │   ├── Planner Logics/       — PlannerRuleBase, PlannerFunctionNode
│   │   │   │   ├── Shape Generators/ — SG_ генераторы форм
│   │   │   │   └── Utilities/        — Базовые классы
│   │   │   └── Planner Graph/        — PGGPlanner_NodeBase, PE_Start
│   │   ├── Mod Graph/                — MR_ ноды (40 файлов)
│   │   ├── Rules Logics/             — SR_ правила (~85 файлов)
│   │   └── Editor/                   — Редакторские окна и инспекторы
│   ├── Tile Designer/                — Генерация архитектурных мешей
│   ├── Object Stamper/               — Штамповка объектов
│   ├── Pipe Generator/               — Генерация труб
│   ├── Generators Shared/            — Общие утилиты генерации
│   └── Mesh Operations CSG/          — CSG-операции с мешами
├── Shared Tools/
│   └── Node FGraph/                  — Ядро нодового графа
│       ├── Node Base/                — FGraph_NodeBase
│       ├── Node Elements/            — NodePortBase, [Port]
│       └── Node Port Types/          — BoolPort, IntPort, FloatPort
└── Plugins - Shared/
    ├── FHelpers/                     — Общие хелперы
    └── Math Helpers/                 — Математические утилиты
```
