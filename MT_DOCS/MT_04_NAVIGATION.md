# MT_04_NAVIGATION.md

> **Навигация по декомпилу и чек-листы задач.**
> Правила и факты — `MT_02_RULES`. Симптомы и паттерны — `MT_03_KNOWLEDGE`.
> Build 42.21. **История:** `MT_DOCS/CHANGELOG.md`.

## 1. Источник истины по API

**Декомпил 42.21** — два слоя, оба в репозитории проекта:

**Java-слой:** `Java_Decompile/zombie/`
**Дерево Java:** `Java_Decompile/Java_Decompile_tree.md`
- Открывается за секунды, сгруппировано по пакетам.
- Содержит: `ItemContainer`, `IsoPlayer`, `IsoGameCharacter`, `WornItems`, `CraftRecipeData`, `FluidContainer`, MP-пакеты и т.д.

**Lua-слой:** `Lua_Decompile/lua/`
**Дерево Lua:** `Lua_Decompile/Lua_Decompile_tree.md`
- Содержит: `ISWearClothing`, `ISTimedActionQueue`, `ISBaseTimedAction`, `ISInventoryTransferAction`, `ISAttachItemHotbar`, `ISUnequipAction`, `ISRemoveItemAction`, `ISMoveableCursor`, `ISMoveableInfoWindow` и все остальные `IS*Action` / `IS*Panel`.

**Подтверждённые пути ключевых Lua-файлов:**
| Файл | Полный путь |
|---|---|
| `ISWearClothing.lua` | `lua/shared/TimedActions/ISWearClothing.lua` |
| `ISTimedActionQueue.lua` | `lua/client/TimedActions/ISTimedActionQueue.lua` |
| `ISInventoryTransferAction.lua` | `lua/client/TimedActions/ISInventoryTransferAction.lua` |
| `ISAttachItemHotbar.lua` | `lua/shared/TimedActions/ISAttachItemHotbar.lua` |
| `ISMoveableCursor.lua` | `lua/server/BuildingObjects/ISMoveableCursor.lua` |
| `ISMoveableInfoWindow.lua` | `lua/client/ISUI/ISMoveableInfoWindow.lua` |

**Структура Lua-слоя** (проверять по `Lua_Decompile_tree.md`):
- `shared/TimedActions/` — общие timed-actions (`ISWearClothing`, `ISAttachItemHotbar`).
- `client/TimedActions/` — клиентские timed-actions (`ISTimedActionQueue`, `ISInventoryTransferAction`).
- `client/ISUI/` — UI-панели и окна (`ISMoveableInfoWindow`).
- `server/BuildingObjects/` — серверные объекты (`ISMoveableCursor`).
- `client/`, `shared/`, `server/` — остальное по логике.

**Правила запроса факта:**
1. **Определи слой** перед запросом:
   - **Java-API** (`ItemContainer`, `IsoPlayer`, `WornItems`, `CraftRecipeData`, `FluidContainer`) → `Java_Decompile/zombie/<полный/путь/Файл>.java`
   - **Lua-API** (`IS*Action`, `IS*Panel`, `ISBaseTimedAction`, `ISWearClothing`, `ISTimedActionQueue`) → `Lua_Decompile/lua/<client|shared|server>/.../<Файл>.lua`
2. Просить **конкретный файл** (`ItemContainer.java`, `ISWearClothing.lua`), не «весь декомпил».
3. Указывать **полный путь** (`zombie/inventory/ItemContainer.java`, `lua/shared/TimedActions/ISWearClothing.lua`) — одноимённые файлы бывают в разных пакетах.
4. Указывать строку в ответе (`ItemContainer.java:215`, `ISWearClothing.lua:42`).
5. **Билды 41 и 42 различаются, не путать.**

**Репозиторий декомпила:**
- Версия игры: `42.21`
- Декомпилятор: Vineflower (Zomboid Decompiler v0.3.2)
- Слои: Java (`Java_Decompile/`) + Lua (`Lua_Decompile/`) — оба полные.
- **Назначение:** справочник по внутренностям движка для Lua-модов и Java-патчей через **ZombieBuddy**.

---

## 2. Структура `zombie/` (Java-слой)

```
zombie/
├── characters/        ← IsoPlayer, IsoZombie, IsoGameCharacter
│   ├── WornItems/     ← WornItem, WornItems, BodyLocation
│   ├── AttachedItems/ ← AttachedItem, AttachedItems
│   └── action/        ← ActionContext, ActionGroup
├── inventory/         ← ItemContainer, InventoryItem
│   ├── types/         ← HandWeapon, Food, Clothing, Drainable
│   └── recipemanager/ ← ItemRecipe, RecipeMonitor
├── entity/components/ ← ECS-компоненты
│   ├── crafting/      ← CraftRecipeData, CraftLogic, HandcraftLogic
│   ├── fluids/        ← FluidContainer, Fluid
│   └── ...
├── iso/               ← IsoWorld, IsoCell, IsoGridSquare
├── network/packets/   ← MP-пакеты
│   ├── world/         ← AddInventoryItemToContainerPacket, SyncItemFieldsPacket
│   ├── character/     ← PlayerPacket, ZombiePacket
│   └── ...
├── scripting/objects/ ← Item, ItemBodyLocation, TimedActionScript
├── ai/states/         ← ZombieIdleState, PathFindState
├── popman/            ← ZombiePopulationManager
└── ...
```

---

## 3. Cheat sheet путей (по темам)

### 3.1. Инвентарь и контейнеры

| Что ищем | Полный путь |
|---|---|
| `AddItem`, `Remove`, `getItems` | `zombie/inventory/ItemContainer.java` |
| `getBodyLocation`, `getItemContainer`, `getFluidContainer`, `IsInventoryContainer` | `zombie/inventory/InventoryItem.java` |
| Фабрика предметов | `zombie/inventory/InventoryItemFactory.java` |
| Спавн лута | `zombie/inventory/ItemPickerJava.java` |
| Починка предметов | `zombie/inventory/FixingManager.java` |
| Типы предметов (`HandWeapon`, `Food`, `Clothing`) | `zombie/inventory/types/` |
| Рецепты (`ItemRecipe`) | `zombie/inventory/recipemanager/` |

### 3.2. Worn/Attached предметы

| Что ищем | Полный путь |
|---|---|
| Worn-контейнер | `zombie/characters/WornItems/WornItems.java` |
| Отдельный надетый предмет (`getLocation`) | `zombie/characters/WornItems/WornItem.java` |
| Слоты тела | `zombie/characters/WornItems/BodyLocation.java`, `BodyLocations.java`, `BodyLocationGroup.java` |
| Хотбар (attached) | `zombie/characters/AttachedItems/AttachedItem.java`, `AttachedItems.java` |

### 3.3. Персонаж

| Что ищем | Полный путь |
|---|---|
| `IsoPlayer` (worn/attach/anim методы) | `zombie/characters/IsoPlayer.java` |
| База персонажа | `zombie/characters/IsoGameCharacter.java` |
| Живой персонаж | `zombie/characters/IsoLivingCharacter.java` |
| Зомби | `zombie/characters/IsoZombie.java` |
| Группа зомби | `zombie/characters/ZombieGroup.java` |
| Профессии/скиллы | `zombie/characters/professions/`, `zombie/characters/skills/` |

### 3.4. Крафт

| Что ищем | Полный путь |
|---|---|
| `craftRecipeData` из `OnCreate` | `zombie/entity/components/crafting/recipe/CraftRecipeData.java` |
| Менеджер рецептов | `zombie/entity/components/crafting/recipe/CraftRecipeManager.java` |
| Handcraft-логика | `zombie/entity/components/crafting/recipe/HandcraftLogic.java` |
| Output/Input мапперы | `zombie/entity/components/crafting/recipe/OutputMapper.java`, `InputItemNode.java` |
| Скрипт-обёртка | `zombie/scripting/entity/components/crafting/CraftRecipe.java` |

### 3.5. Жидкости

| Что ищем | Полный путь |
|---|---|
| `copyFluidsFrom`, `Empty`, `addFluid` | `zombie/entity/components/fluids/FluidContainer.java` |
| Тип жидкости | `zombie/entity/components/fluids/Fluid.java`, `FluidInstance.java`, `FluidType.java` |
| Параметры | `zombie/entity/components/fluids/FluidProperties.java` |

### 3.6. MP-синхронизация

| Что ищем | Полный путь |
|---|---|
| Добавление в контейнер | `zombie/network/packets/world/AddInventoryItemToContainerPacket.java` |
| Удаление из контейнера | `zombie/network/packets/world/RemoveInventoryItemFromContainerPacket.java` |
| Синхронизация полей предмета | `zombie/network/packets/world/SyncItemFieldsPacket.java` |
| Синхронизация ModData | `zombie/network/packets/world/SyncItemModDataPacket.java` |
| Общий инвентарь | `zombie/network/packets/world/AddItemInInventoryPacket.java` |

### 3.7. Скрипт-объекты

| Что ищем | Полный путь |
|---|---|
| Базовый `Item` из скриптов | `zombie/scripting/objects/Item.java` |
| `ItemBodyLocation` | `zombie/scripting/objects/ItemBodyLocation.java` |
| `TimedActionScript` | `zombie/scripting/objects/TimedActionScript.java` |
| `ContainerType` | `zombie/scripting/objects/ContainerType.java` |
| `AttachmentType` | `zombie/scripting/objects/AttachmentType.java` |

### 3.8. Зомби и AI

| Что ищем | Полный путь |
|---|---|
| `ZombieIdleState` | `zombie/ai/states/ZombieIdleState.java` |
| `ZombieGroupManager` | `zombie/ai/ZombieGroupManager.java` |
| `PathFindState` | `zombie/ai/states/PathFindState.java` |
| `PathFindBehavior2` | `zombie/pathfind/PathFindBehavior2.java` |
| `ZombiePopulationManager` | `zombie/popman/ZombiePopulationManager.java` |

---

## 4. Чек-листы задач

### 4.1. Добавить новый предмет (item script)

1. Создать `media/scripts/<категория>/MT_<Имя>.txt` с блоком `module Base { item MT_Имя { ... } }`.
2. Указать `Type`, `DisplayName`, `Icon`, `Weight`, при необходимости — `BodyLocation`, `CanBeEquipped`, `AttachmentType`.
3. Локализация — ключ `ItemName_MT_Имя` в `Translate/{EN,RU}/ItemName.json`. **Trailing comma обязательна.**
4. Иконка — `media/textures/item_MT_<Имя>.png` (256×256).
5. Модель (если нужна) — `media/models_X/<категория>/`.
6. Проверить: предмет появляется в Debug Items List (`F6`), иконка отображается, `DisplayName` переводится.

### 4.2. Добавить рецепт крафта

1. Открыть `media/scripts/<категория>/MT_<Имя>_Recipie.txt` (или создать).
2. Блок:
   ```
   craftRecipe TBag1 {
       timedAction = SewingCloth,
       Time = 200,
       xpAward = Tailoring:1,
       AllowBatchCraft = False,
       Tags = AnySurfaceCraft,
       category = [MT]_Bags,
       inputs { ... }
       outputs { ... }
   }
   ```
3. В `inputs` — предметы с `mode:destroy` (расходуются) или `mode:keep` (нужны, но остаются).
4. В `outputs` — что создаётся.
5. **Если рецепт заменяет предмет с содержимым** — использовать §4.3 (паттерн TBag).
6. Проверить: рецепт появляется в меню крафта, все inputs корректно потребляются/сохраняются.

### 4.3. Заменить предмет в крафте с переносом содержимого

**Полный паттерн — `MT_03_KNOWLEDGE` §3.1.** Краткий чек-лист:

- [ ] В `OnCreate`: собрать `newItem`, `oldItem`, `wasWorn`/`hotbarIndex` (если старый надет/на хотбаре).
- [ ] В `OnCreate`: снять старый (если надет), **не трогая контейнер**.
- [ ] В `OnCreate`: положить в `Pending` с `wait = 5`, `ticks = 60`.
- [ ] В `OnTick` (после `wait`): перенести вещи через `AddItem` (без явного `Remove`).
- [ ] В `OnTick`: перенести жидкости через `copyFluidsFrom`.
- [ ] В `OnTick`: надеть через `ISWearClothing`, прикрепить через `attachItem`.
- [ ] В `OnTick`: `syncItemFields` если существует.
- [ ] `ticks--` каждый тик; при `ticks <= 0` — `table.remove`.
- [ ] Протестировать в SP: сумка с вещами + надетая + с водой во фляге.

### 4.4. Добавить timed action (IS-действие)

1. Создать `<Категория>/IS<Имя>Action.lua` (в `media/lua/client/`).
2. Наследование: `ISBaseTimedAction` (или `ISWearClothing` / `ISInventoryTransferAction`, если подходит).
3. Методы: `:new(character, ...)`, `:isValid()`, `:start()`, `:update()`, `:stop()`, `:perform()`, `:getDuration()`.
4. В `:start()` — `self:setActionAnim(...)`, `self.character:reportEvent(...)`.
5. В `:perform()` — фактическое действие (перенос, надевание и т.п.).
6. Вызов: `ISTimedActionQueue.add(IS<Имя>Action:new(char, ...))`.
7. **Не забыть**: `if IS<Имя>Action and ISTimedActionQueue then` перед вызовом — как fallback для старых сборок.

### 4.5. Добавить Java-патч (ZombieBuddy)

1. Создать `<Мод>/java/<пакет>/Patch_<Класс>.java`.
2. Импорт: `me.zed_0xff.zombie_buddy.Patch`.
3. Аннотация: `@Patch(className = "zombie.path.Class", methodName = "method")`.
4. **Правильный способ** — advice-стиль: `@Patch.OnEnter` / `@Patch.OnExit` (см. `MT_03_KNOWLEDGE` §3.4).
5. **`@Patch.Replace` отсутствует** в ZombieBuddy (см. `MT_03_KNOWLEDGE` §2.2).
6. **Все поля/методы — `public`** (иначе `IllegalAccessError`).
7. Для работы с зомби — **закомментировать `isRemoteZombie()`** (см. `MT_03_KNOWLEDGE` §2.3).
8. Для `PZ_Optimization` — `LongAdder` для счётчиков, `volatile float[]` для снимка.
9. `DEBUG = false` в релизной сборке.
10. Проверить: `JAVA_HOME` установлен, консоль перезапущена, `Found patch class` в логе.

### 4.6. Обновить декомпил при патче игры

1. Скачать новую версию игры (Steam).
2. Запустить `Zomboid Decompiler v0.3.2` (Vineflower) на `zombie.jar`.
3. Обновить `Java_Decompile/zombie/` и `Lua_Decompile/lua/` (структура должна совпадать с оригиналом).
4. Перегенерировать `Java_Decompile_tree.md` и `Lua_Decompile_tree.md` (команды — в `*_Decompile_commands_for_tree/`).
5. **Сверить с `MT_02_RULES` / `MT_03_KNOWLEDGE`** — если API изменилось, обновить правила/факты.
6. Запись в `MT_DOCS/CHANGELOG.md` — «обновлён декомпил до версии X», «API Y изменилось, правило в §Z обновлено».

### 4.7. Отладка через Lua-консоль

1. Открыть в игре: `Esc` → Debug → Lua Console (или `F11` → Lua).
2. Выполнить код. Примеры:
   - Установить ядро: `local p = getPlayer(); local w = MT_SmartWatch.Watch.getEquippedWatch(p); MT_SmartWatch.OSCore.installCore(w, p:getInventory():getFirstType("Base.MTSW_Core_042"))`
   - Проверить предмет: `getPlayer():getInventory():getFirstType("Base.TBag2")`
3. `print()` — только ASCII (см. `MT_02_RULES` §2.1).
4. Полный лог — `Zomboid/console.txt`.
5. Для UI-панелей — `PZDebugWindow` (`F11`) → Inspect.

---

## 5. Быстрые ссылки

| Что | Где |
|---|---|
| Декомпил 42.21 | `Java_Decompile/` + `Lua_Decompile/` (в репозитории проекта) |
| PZ Wiki Lua Events | `https://pzwiki.net/wiki/Lua_Event` |
| ZombieBuddy | `me.zed_0xff.zombie_buddy.Patch` |
| `[MT] SmartWatch` repo | `https://github.com/yatsureru/-MT-SmartWatch` |