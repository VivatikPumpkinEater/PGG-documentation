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

## 6. Build Planner Nodes PR
### Ноды планировщика

**178 нод** в `PGG/Planners Related/Planner Coded Nodes/`

### 6.1 Math (35 нод)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Add** | PR_Add | Сложение значений. С полями — объединение в список |
| **Subtract** | PR_Subtract | Вычитание. С полями — убирает поля из списка |
| **Multiply** | PR_Multiply | Умножение. С полями — создаёт список полей |
| **Divide** | PR_Divide | Деление |
| **Half Value** | PR_Half | Делит значение на 2 |
| **Always Positive (Abs)** | PR_ABS | Модуль (абсолютное значение) |
| **Invert Value** | PR_Invert | Инверсия значения (-1 * v) |
| **One Minus** | PR_OneMinus | 1 − значение |
| **Clamp** | PR_Clamp | Ограничение значения диапазоном |
| **Round Value** | PR_Round | Округление |
| **Round Position** | PR_RoundPosition | Округление позиции до сетки |
| **Lerp** | PR_Lerp | Линейная интерполяция |
| **Normalize** | PR_Normalize | Нормализация вектора |
| **Magnitude** | PR_Magnitude | Длина вектора / расстояние |
| **Dot / Angle Product** | PR_DotProduct | Скалярное произведение / угол между векторами |
| **Cross** | PR_Cross | Векторное произведение (X↔Z) |
| **Append** | PR_Append | Собрать X, Y, Z в Vector3 |
| **Split** | PR_Split | Разобрать значение на компоненты |
| **Get X/Y/Z** | PR_V3Get | Получить X/Y/Z компоненту Vector3 |
| **Get Value by Axis** | PR_GetValueByAxis | Получить компоненту по оси |
| **Dominant Axis** | PR_DominantAxis | Доминантная ось вектора |
| **Angle Direction** | PR_AngleDirection | Угол направления |
| **Rotate Direction** | PR_RotateDirection | Поворот направления |
| **Dir to Rot** | PR_DirectionToRotation | Направление → эйлеры (Look Direction) |
| **Rot to Dir** | PR_RotationToDirection | Эйлеры → направление |
| **Wrap Angle** | PR_WrapAngle | Нормализация угла (0–360 или ±180) |
| **Get Random** | PR_GetRandom | Случайное число в диапазоне |
| **Get Random** (with inputs) | PR_GenerateRandom | Случайное число с входными параметрами |
| **Choose Random** | PR_ChooseRandom | Случайный выбор из нескольких входов |
| **Choose Greater / Smaller** | PR_MathMinMax | Min/Max |
| **Period % Mod** | PR_PeriodModulo | Периодический modulo (%) |
| **Trigger** | PR_BoolTrigger | Триггер boolean значения |
| **To Bool** | PR_ToBool | Конвертация значения в bool |
| **Value** | PR_Value | Вывод значения произвольного типа |
| **Null Value** | PR_Null | Вывод null |

### 6.2 Cells — Работа с ячейками (30 нод)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Iterate Cells** | PR_IterateCells | Цикл по всем ячейкам поля |
| **Iterate Cells In Line** | PR_IterateCellsInLine | Итерация ячеек по линии |
| **Iterate Cell Outside Dir** | PR_IterateCellOutsideDirections | Итерация по внешним направлениям ячейки |
| **Get Cell Position** | PR_GetCellPosition | Получить мировую позицию ячейки |
| **Get Cell** | PR_GetCell | Получить ячейку по мировой позиции |
| **Get Cell By Index** | PR_GetCellByIndex | Получить ячейку по индексу |
| **Get Nearest Cell** | PR_GetNearestCell | Ближайшая ячейка к позиции |
| **Get Nearest Cell In** | PR_GetNearestCellIn | Ближайшая ячейка в наборе |
| **Get Random Cell In** | PR_GetRadomCellIn | Случайная ячейка из набора |
| **Get Random Cell With No Instruction** | PR_GetRadomCellWithNoInstruction | Случайная ячейка без инструкций |
| **Get Most Middle Cell** | PR_GetMostMiddleCell | Самая центральная ячейка |
| **Most Cells In Dir** | PR_GetMostCellsDirection | Направление с наибольшим числом ячеек |
| **Select Cells On Edge** | PR_SelectCellsOnEdge | Выбрать ячейки на краю поля |
| **Get Colliding / Not Colliding Cells** | PR_GetCollidingCells | Ячейки, пересекающиеся с другим полем |
| **Is Any Cell In Position** | PR_IsAnyCellInPosition | Есть ли ячейка в позиции? |
| **Detect Cell in** | PR_DetectCellIn | Детектировать ячейку в позиции |
| **Is cell diagonal to** | PR_AreCellsDiagonalToEach | Проверка диагональности ячеек |
| **Contact In Direction** | PR_CheckContactInDirection | Контакт с полем в направлении |
| **Bounds Sweep Collision** | PR_BoundsSweepCollisionInDirection | Sweep-коллизия bounds в направлении |
| **Get Cell Aligned Position** | PR_RoundAccordingly | Выровнять позицию под сетку другого поля |
| **Set/Add Cell Parameter** | PR_SetCellParameter | Установить/добавить параметр ячейки |
| **Cell Parameters** | PR_GetCellParameters | Прочитать параметры ячейки |
| **Instruction Parameters** | PR_GetCellInstructionParams | Параметры инструкции ячейки |
| **Add Cell Instruction** | PR_AddCellInstruction | Добавить инструкцию ячейке |
| **Copy Cell Datas** | PR_CopyOtherCellParams | Копировать инструкции/данные из другой ячейки |
| **Cell Contains Data String** | PR_CellContainsDataString | Проверить наличие строки в данных ячейки |
| **Check cell data in range** | PR_IsCellDataInRange | Проверка диапазона данных ячейки |
| **Get Direction from to** | PR_DirectionFromTo | Направление от/к позиции или ячейке |
| **Cell Debug** | PR_CellDebug | Отладочная информация ячейки |

### 6.3 Field Planner — Операции с полем (67 нод)

#### Доступ к данным поля (ReadData)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Grid Boundary (Cells)** | PR_GetFieldGridBoundaries | Границы сетки поля |
| **Get Cells Count** | PR_CellsCount | Количество ячеек |
| **Get Bounds** | PR_GetFieldBounds | Bounds поля (несколько полей → один Bounds) |
| **Bounds Parameter** | PR_GetBoundsParameter | Параметр bounds (размер, центр и т.д.) |
| **Get Parameter** | PR_GetFieldParameter | Параметры планировщика |
| **Cell Instructions Count** | PR_GetCellInstructionCount | Количество инструкций ячейки |
| **Get Cell Instruction** | PR_GetCellInstruction | Получить конкретную инструкцию ячейки |
| **Get Field** | PR_GetFieldSelector | Получить FieldPlanner из селектора |
| **Get Fields Count** | PR_FieldsCount | Общее число полей |
| **Instances Count** | PR_FieldInstancesCount | Число экземпляров планировщика |
| **Get Sub-Fields Count** | PR_SubFieldsCount | Число подполей |
| **Get Sub Field** | PR_GetSubField | Получить подполе по индексу |
| **Get Sub Fields of** | PR_GetSubFields | Получить все подполя |
| **Get Field Instance** | PR_GetFieldInstance | Экземпляр планировщика по ID |
| **Get Field** | PR_GetFieldFromInt | Планировщик по числовому индексу |
| **Get Index Of Instance** | PR_GetGlobalIndexOfInstance | Глобальный индекс экземпляра |
| **Instance of Planner** | PR_GetPlannerDuplicate | Дубликат планировщика |
| **Get Field Instances** | PR_GetFieldDuplicates | Все дубликаты |
| **Get Cell Size** | PR_GetCellSize | Размер ячейки сетки |
| **Cell Size Multiply** | PR_ScaleWithFieldCellSize | Масштабирование с учётом размера ячейки |

#### Поиск соседей и других полей

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Neightbour Field** | PR_GetNeightbourField | Ближайшее смежное поле |
| **Choose Neightbour Fields** | PR_ChooseNeightbourFields | Выбрать несколько смежных полей |
| **Nearest Field Planner** | PR_GetNearestFieldPlanner | Ближайший планировщик (с условием) |
| **Farthest Field Planner** | PR_GetFarthestFieldPlanner | Самый дальний планировщик |
| **Get Random Field Instance** | PR_GetRadomFieldPlanner | Случайный планировщик |

#### Проверки полей

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Is Colliding With** | PR_IsFieldCollidingWith | Пересекается ли поле с другим |
| **Is Fully Contained By** | PR_IsFullyContainedBy | Полностью ли содержится внутри другого |
| **Is Aligning With** | PR_IsFieldAligningWith | Выровнено ли поле с другим |
| **Count Alignment Cells** | PR_CountAlignmentsWith | Число выравниваний с другим полем |
| **Bounds Collision** | PR_CheckBoundsCollisionBetween | Коллизия bounds между полями |

#### Модификация формы

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Join Field-Shape Cells** | PR_JoinShapeCells | Объединить ячейки формы |
| **Remove Field Cells** | PR_RemoveFieldCells | Удалить ячейки из поля |
| **Remove All Field Cells** | PR_RemoveAllFieldCells | Удалить все ячейки |
| **Replace Field Cells** | PR_ReplaceFieldCells | Заменить ячейки |
| **Subtract Shape Cells** | PR_ShapeSubtract | Вычесть одну форму из другой |
| **Union Shape Cells** | PR_ShapeUnion | Объединение форм |
| **Split Field** | PR_SplitField | Разделить поле |
| **Each Y Level To Shape** | PR_YToFields | Y-координаты в отдельные поля |
| **Get Disconnected chunks** | PR_GetDisconnectedChunks | Найти изолированные группы ячеек |
| **Rectangle Divide** | PR_RectangleDivide | Разделить прямоугольную область |
| **Generate Empty Shape** | PR_GenerateEmptyShape | Пустой контейнер формы |
| **Add Sub Field** | PR_AddSubField | Добавить подполе |
| **Assign Sub Field Param** | PR_AssignSubFieldParameter | Назначить параметр подполю |
| **Remove Too Far Cells** | PR_RemoveTooFarCells | Удалить слишком далёкие ячейки |
| **Discard Field** | PR_DiscardField | Отбросить поле целиком |

#### Трансформация поля (WholeFieldPlacement)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Field Position** | PR_GetFieldPosition | Получить позицию поля |
| **Set Field Position** | PR_SetFieldPosition | Установить позицию |
| **Round Field Position** | PR_RoundFieldPosition | Округлить позицию поля |
| **Get Field Rotation** | PR_GetFieldRotation | Получить ротацию поля |
| **Set Field Rotation** | PR_SetFieldRotation | Установить ротацию |
| **Set Field Origin** | PR_SetFieldOrigin | Установить origin поля |
| **Center Field Origin** | PR_CenterFieldOrigin | Центрировать origin |

#### Размещение и коллизии

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Tight Placement** | PR_TightPlacement | Плотное размещение (прижать к другому полю) |
| **Customized Tight Placement** | PR_CustomTightPlacement | Кастомное плотное размещение |
| **Push Out Collision** | PR_PushOutOfCollision | Вытолкнуть из коллизии |
| **Push Out Away** | PR_PushOutAway | Вытолкнуть подальше |
| **Push Until not Collides** | PR_PushInDirUntilNotCollides | Толкать пока не перестанет пересекаться |
| **Push Until Contained By** | PR_PushInDirUntilFullyContained | Толкать пока полностью не войдёт |
| **Push For Align** | PR_PushInDirToAlign | Толкать для выравнивания |
| **Bounds Separate Push** | PR_BoundsSeparatePushOut | Разделить пересекающиеся bounds |
| **Align Self To** | PR_AlignTo | Выровнять к другому полю |
| **Quick Align** | PR_QuickAlign | Быстрое выравнивание |

#### Переменные

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Local Variable** | PR_GetLocalVariable | Прочитать локальную переменную |
| **Set Local Variable** | PR_SetLocalVariable | Установить локальную переменную |
| **Set Local Variable Allocated** | PR_SetLocalVariableAlloc | С сохранением между итерациями |
| **Get Internal Value** | PR_GetInternalValueVariable | Внутренняя переменная экземпляра |
| **Set Internal Value** | PR_SetInternalValueVariable | Записать внутреннюю переменную |
| **Get Field Variable** | PR_GetFieldPlannerVariable | Переменная планировщика |

### 6.4 Logic — Логика и ветвление (13 нод)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **If == Switch** | PR_EqualIfSwitch | Сравнение → Output Value |
| **If A==B : False/True** | PR_EqualIfExecution | Сравнение → Execute A/B |
| **If => Execute A or B** | PR_EqualIfSwitchExecution | Bool → Execute A/B |
| **Output A or B** | PR_BoolIfSwitch | If true/false → Output A or B |
| **If A==B : False/True** | PR_TrueFalseSwitch | Сравнение → return True/False |
| **If => Return Bool** | PR_BoolAccumulate | Накопление условий → bool |
| **If Between : False/True** | PR_ValueBetweenSwitch | Значение в диапазоне |
| **If => 50% Execute A or B** | PR_5050Execution | 50% вероятность → ветка |
| **Is Null?** | PR_IsNull | Проверка на null |
| **Iterator (Loop)** | PR_IteratorLoop | Цикл-итератор |
| **Iterate Rotations** | PR_RotorLoop | Перебор поворотов (0°, 90°, 180°, 270°) |
| **Iterate List** | PR_IterateList | Цикл по списку object |

### 6.5 Build Setup (11 нод)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Collect All Fields** | PR_CollectFields | Собрать все поля в список |
| **Choose Fields** | PR_ChooseFields | Выбрать поля по условию |
| **Choose One Field** | PR_ChooseOneField | Найти одно поле по условию |
| **Iterate Fields** | PR_IterateFields | Цикл по полям |
| **Iteration Index** | PR_GetIterationIndex | Текущий индекс итерации |
| **Build area Bounds** | PR_GetBuildAreaBounds | Bounds области генерации |
| **Get Build Variable** | PR_GetBuildVariable | Переменная Build Planner |
| **Expand Bounds Size** | PR_ExpandBoundsSize | Расширить bounds |
| **Generate Bounds** | PR_GenerateBounds | Сгенерировать bounds |
| **Schedule Field Injection** | PR_ScheduleFieldInjection | Инъекция переменной в FieldSetup |
| **Call Graph (experimental)** | PR_CallGraph | Вызвать другой подграф |

### 6.6 Generating — Генерация форм (10 нод)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Rect Generate** | PR_RectGenerate | Прямоугольник заданных размеров |
| **Line Generate** | PR_LineGenerate | Линия/коридор |
| **Path Find (deprecated)** | PR_PathFindGenerate | Поиск пути A* (устаревший) |
| **Path Find Generate** | PR_FindPathTowards | Pathfind к целевой позиции |
| **Bounds To Cells** | PR_BoundsToCells | Конвертировать bounds в ячейки |
| **Add Cell to Field** | PR_AddCellToField | Добавить ячейку к полю |
| **Get Inline Shape** | PR_GetInlineShape | Внутренний контур формы |
| **Get Outline Shape** | PR_GetOutlineShape | Внешний контур формы |
| **Get Fattened Shape** | PR_GetFattenShape | Утолстить тонкие пути |
| **Convert To New Scale** | PR_GetScaleConvertedShape | Конвертировать форму с другим масштабом |

### 6.7 Utilities & Debug (8 нод)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| *(текст заголовка)* | PR_Comment | Комментарий на графе |
| *(текст группы)* | PR_Group | Группировка нод |
| **Console Log** | PR_ConsoleLog | Debug.Log |
| **Debug Field Highlight** | PR_DebugDrawFieldHighlight | Визуализация поля в Scene View |
| **Debug Draw Position** | PR_DebugDrawPosition | Визуализация позиции |
| **To String** | PR_ToString | Конвертация в строку |
| **Redirect Value** | PR_OneCallFilter | Фильтр однократного вызова |
| **Rewire** | PR_Rewire | Переподключение execution-потока |

### 6.8 Specific Solutions (2 ноды)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Generate Fields Connections** | PR_PathFind_DeloneConnections | Связи комнат по Delaunay (Bowyer–Watson) |
| **Perlin Noise Offset** | PR_ApplyPerlinNoiseOffset | Смещение высоты по Perlin Noise |

### 6.9 Community (2 ноды)

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Remove Cells Under** | PR_RemoveCellsUnder | Удалить ячейки «под» другими |
| **Path Find - Lower Cost On (custom)** | PR_PathFindGenerateLowerCost | Pathfind с пониженной стоимостью |

---

## 7. Mod Graph Nodes MR
### Ноды модификации

**40 нод** в `PGG/Mod Graph/`  
Используются в Mod Node Graph внутри FieldSetup для управления спавном на уровне отдельных ячеек.

### 7.1 Управление спавном

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Restore Allow Spawn** | MR_AllowSpawn | Разрешить спавн (восстановить после Break) |
| **Break Spawner** | MR_BreakSpawner | Запретить спавн |
| **Disable Main Spawn** | MR_DisableSpawningMainPrefab | Отключить спавн основного префаба |
| **Generate Spawn** | MR_GenerateSpawn | Создать новые данные спавна |
| **Apply Prefab** | MR_ApplyPrefabToSpawn | Назначить префаб для спавна |
| **Extra Spawn** | MR_AddExtraSpawn | Доп. спавн в очередь ячейки |
| **Copy Spawn** | MR_CopySpawn | Копировать данные спавна |
| **Remove Spawn** | MR_RemoveSpawn | Удалить спавн из очереди |
| **Add Spawn Stigma** | MR_AddSpawnStigma | Строковая стигма спавну |
| **Add Cell Data** | MR_AddCellData | Cell Data строка ячейке |

### 7.2 Позиция / ротация / масштаб

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Spawn Position** | MR_GetPosition | Позиция спавна |
| **Get Spawn Rotation** | MR_GetRotation | Ротация спавна |
| **Get Spawn Scale** | MR_GetScale | Масштаб спавна |
| **Set Spawn Position** | MR_SetPosition | Установить/сместить позицию |
| **Set Spawn Rotation** | MR_SetRotation | Установить/сместить ротацию |
| **Set Spawn Scale** | MR_SetScale | Установить масштаб |
| **Clear Offsets** | MR_ClearOffsets | Сбросить все оффсеты |

### 7.3 Работа с ячейками

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Cell At** | MR_GetCellAt | Ячейка по смещению от текущей |
| **Get Cell Position** | MR_GetCellPosition | Мировая позиция ячейки сетки |
| **Cell State** | MR_GetCellStateOnGrid | Состояние ячейки (в сетке / вне / занята тегом) |
| **Get Cells Around** | MR_GetCellsAround | Ячейки в квадрате вокруг |
| **Get Cell Neighbour** | MR_GetNeighbourCell | Соседняя ячейка (с поворачиваемым смещением) |
| **Get Connected Cells** | MR_GetFillConnectedCells | Все достижимые ячейки (fill, без диагоналей) |
| **Nearest Cell With** | MR_GetNearestCellWith | Ближайшая ячейка с условием |
| **Distance to other cells** | MR_GetOtherCellInDistance | Расстояние до ячеек по пути |
| **Iterate Cells** | MR_IterateCells | Итерация по списку ячеек |
| **Iterate Instructions** | MR_IterateCellInstructions | Итерация по инструкциям ячейки |
| **Line Check Cells** | MR_LineCheckCells | Проверка ячеек по линии в 4 направлениях |

### 7.4 Данные спавна

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Spawn From Cell** | MR_GetSpawnFromCell | Первый/все спавны ячейки |
| **Get Spawns In Cell** | MR_GetSpawnsInCell | Все спавны в ячейке |
| **Get Spawn Prefab** | MR_GetSpawnPrefab | Ссылка на спавнимый префаб |
| **Get Prefab Bounds** | MR_GetPrefabBounds | Bounds префаба |
| **Spawn is Tagged / Stigmed** | MR_SpawnIsTagged | Проверка тега/стигмы спавна |
| **Iterate Spawns** | MR_IterateSpawns | Итерация по спавнам ячейки |

### 7.5 Другое

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Get Field Variable** | MR_GetFieldVariable | Переменная FieldSetup / ModPack |
| **Command Direction** | MR_GetCommandDirection | Направление команды ячейки |
| **Get Grid Cell Size** | MR_GetGridCellSize | Размер ячейки сетки |
| **Grid Size** | MR_GridSize | Размеры сетки |
| **Tile Designer** | MR_TileDesigner | Сгенерировать объект через Tile Designer |

---

## 8. Field Setup Spawn Rules SR
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

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Cell Operation** | SR_CellOperation | Операции над выбранной ячейкой |
| **Analyze Cell** | SR_AnalyzeCell | Проверка состояния ячейки (allow/deny) |
| **Add Cell Data String** | SR_AddCellDataString | Записать cell data при условиях |
| **Cell Spawns Count** | SR_CellSpawnsCount | Условие по числу спавнов в ячейке |
| **Hide Cell** | SR_HideCell | Скрытие ячейки |
| **Prevent Mods Spawns** | SR_PreventModsSpawns | Блокировка спавнов других модификаторов |
| **Prevent Spawns** | SR_PreventSpawns | Блокировка следующих спавнеров по тегам |
| **Remove Spawn** | SR_RemoveSpawn | Удаление спавна |
| **Remove Spawns Tool** | SR_RemoveSpawnsTool | Удаление спавнов по наборам условий |
| **Remove In Position** | SR_RemoveInPosition | Удаление спавнов по расстоянию |
| **Remove In Direction** | SR_RemoveInDirection | Удаление по направлению (Legacy) |

### 8.2 Placement — Условия размещения

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Check cell neightbours** | SR_CellNeightbours | Гибкая проверка соседних ячеек (3D-ротор) |
| **Multi check cell neightbours** | SR_MultiCheckCellNeightbours | Множественные условия по соседям |
| **Grid Position** | SR_CellPosition | Условие по позиции ячейки на сетке |
| **If World Position** | SR_IfWorldPosition | Условие по мировой позиции |
| **If Rotated** | SR_IfRotated | Допуск по диапазону поворота ячейки |
| **If cell contains Tag** | SR_IfCellContainsTag | Теги/стигма/data в ячейке |
| **On Grid Corner** | SR_OnGridCorner | Углы сетки |
| **Allow Spawn Every Few** | SR_AllowEveryFew | Каждые N ячеек |
| **Check cells in line** | SR_CheckCellsInLine | Проверка ячеек по линии |
| **Check Free Space** | SR_FreeSpace | Проверка свободного места (Alpha) |
| **Helper Pivot Correction** | SR_HelperPivotCorrection | Коррекция координат |
| **Simulate Physics** | SR_SimulatePhysics | Физическая симуляция после спавна |

### 8.3 Transforming — Трансформация

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Move-Rotate-Scale** | SR_MoveRotateScale | Базовый оффсет PRS |
| **Get Position-Rotation-Scale** | SR_GetPosRotScale | Получить PRS от другого спавна |
| **Prefab Offset** | SR_PrefabOffset | Оффсет из координат префаба |
| **Conditional Push Position** | SR_ConditionalPushPosition | Push с разными значениями по углу |
| **Perlin Noise Rotation-Scale** | SR_PerlinNoiseRotationOrScale | Perlin Noise для ротации/масштаба |

#### Legacy Transforming

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Scale** | SR_Scale | Масштабирование |
| **World Offset** | SR_WorldOffset | Смещение позиции (без ротации) |
| **Direct Offset** | SR_DirectOffset | Прямое смещение |
| **Push Position** | SR_PushPosition | Аддитивный оффсет |
| **Random Scale Ranged** | SR_ScaleRange | Случайный масштаб в диапазоне |
| **Get Coordinates** | SR_GetCoords | Координаты от спавна по тегу |
| **Get Restricted Rotation** | SR_GetRestrictedRotation | Ротация из command ячейки |
| **Get Command Rotation** | SR_GetCommandRotation | Ротация из command |
| **Change Rotation Pivot** | SR_ChangeRotationPivot | Поворот вокруг другого pivot |

### 8.4 Modelling — Генерация объектов

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Tile Designer** | SR_TileDesigner | Генерация через Tile Designer |
| **Spawn Random Mesh Renderer** | SR_RandomMesh | Mesh Renderer со случайным мешем |
| **Cable Mesh Generator** | SR_CableGenerator | Процедурный кабельный меш |
| **Replace MeshFilter Mesh** | SR_ReplaceFilterMesh | Замена mesh у MeshFilter |
| **Replace Spawn with Random Prefab** | SR_ReplaceWithRandomPrefab | Замена префаба на случайный |
| **Replace Spawned Prefab** | SR_ReplacePrefab | Замена префаба + переменные |
| **Set Mesh Material** | SR_SetMaterial | Назначение материала |
| **Set Random Mesh Material** | SR_RandomMaterial | Случайный материал из списка |
| **Set Game Object Layer** | SR_SetGameObjectLayer | Установка layer у GameObject |
| **Acquire Spawn** | SR_AcquireSpawn | Временный спавн для «пустого» слота |
| **Shedule Mesh Combine** | SR_CombineMesh | Отложенное объединение мешей |
| **Volume Indicator** | SR_VolumeIndicator | Маркер объёма |

#### Roof Generator

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Roof Plane Generator** | SR_RoofPlaneGenerator | Генерация плоскости крыши |
| **Roof Side Wall Generator** | SR_RoofSideGenerator | Боковые стены крыши |
| **Roof Generator - Placer** | SR_RoofGeneratorPlacer | Размещение элементов крыши |
| **Roof Generator - Edges Placer** | SR_RoofGeneratorEdgesPlacer | Рёбра/гребень крыши |

### 8.5 Quick Solutions — Быстрые решения

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Wall Placer** | SR_WallPlacer | Размещение стен с выравниванием |
| **Floor Placer** | SR_FloorPlacer | Размещение полов |
| **Doorway Placer** | SR_DoorwayPlacer | Дверные проёмы из guide |
| **Double Doors Helper** | SR_DoubleDoorsHelper | Двойные двери → один проём |
| **Command Dir vs Rotation (Corner Doorway Check)** | SR_CornerDoorwayCheck | Команда vs поворот угла |
| **Stairs Placer Helper** | SR_StairsHelper | Лестницы |
| **Edges Placer** | SR_EdgesPlacer | Плитки по краям |
| **Directional Remove** | SR_DirectionalRemove | Удаление по смещённой позиции |
| **Call Rules Logics Of** | SR_CallRulesOf | Вызов правил другого спавнера |
| **Mod Node Graph** | SR_ModGraph | Логика через Mod Node Graph |
| **Sub Spawner (Deprecated)** | SR_SubSpawner | Доп. спавн (deprecated) |
| **Cables Spawner (Deprecated)** | SR_CablesSpawner | Кабели (deprecated) |

### 8.6 Collision — Коллизии

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Spawn Until Collides** | SR_SpawnUntilCollides | Спавн до коллизии |
| **Check World Collision** | SR_CheckWorldCollision | Проверка коллизий со сценой |
| **Bound Collision Offset** | SR_BoundCollisionOffset | Оффсет по bounds + коллизии (Legacy) |
| **Offset From Bounds** | SR_OffsetFromBounds | Оффсет от bounds другого объекта (Legacy) |

### 8.7 PostEvents — Пост-обработка

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Flatten Terrain Ground** | SR_FlattenTerrain | Выравнивание Terrain |
| **Cut Terrain Hole** | SR_CutTerrainHole | Дыра в Terrain |
| **Remove Terrain Trees** | SR_RemoveTerrainTrees | Удаление деревьев Terrain |
| **Remove Terrain Details** | SR_RemoveTerrainDetail | Удаление деталей Terrain |

### 8.8 Operations — Операции

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Stack Spawner** | SR_StackSpawner | Стек префабов (stamper) |
| **Pipe Spawner** | SR_PipeSpawner | Pipe Generator без отдельного пресета |
| **Duplicate Spawns** | SR_DuplicateSpawns | Дублирование спавнов |

### 8.9 Count — Ограничения

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Spawning Probability** | SR_SpawningPropability | Вероятность спавна |
| **Limit Spawning Count** | SR_LimitSpawnCount | Лимит числа спавнов |

### 8.10 FieldAndGrid — Поле и сетка

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| **Grid Specifics** | SR_GridSpecifics | Условия по специфике сетки |
| **Is same Field Setup** | SR_IsSameFieldSetup | Проверка Field Setup |
| **Correct to Center** | SR_OffsetCenter | Смещение к центру сетки |
| **Compare Variable** | SR_CompareVariable | Сравнение переменной |
| **Shift Towards** | SR_ShiftTowards | Сдвиг к цели |

### 8.11 Other

| Имя в Inspector | Класс | Описание |
|-----------------|-------|----------|
| *(текст заголовка)* | SR_Separator | Визуальный разделитель |
| **Comment** | SR_Comment | Комментарий/заметка |
| **Debug Log** | SR_DebugLog | Debug.Log при условиях |

---

## 9. Function Nodes FN
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

## 10. Execution Nodes PE
### Ноды выполнения

| Нода | Описание |
|------|----------|
| **PE_Start** | Начальная нода — точка входа в граф процедур |

`PE_Continue` закомментирован и не активен.

---

## 11. Blueprint Nodes BPNode
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
