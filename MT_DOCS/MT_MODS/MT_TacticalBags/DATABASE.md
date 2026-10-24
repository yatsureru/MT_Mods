# MT_MODS/MT_TacticalBags/DATABASE.md

# [MT] Tactical Bags — DATABASE

> **[MT] AI reminder:** не гадай API — запрашивай декомпил (`MT_04_NAVIGATION` §1). Ищи идиому до примитива. Помечай `[?]` предположения. Полный протокол — `MT_01_AI_PROTOCOL`.
> **Общие правила:** `MT_02_RULES`. **Паттерн рекрафта:** `MT_03_KNOWLEDGE` §3.1.

## 1. Архитектура

### 1.1. Рецепты (`tactical_bags_recipie.txt`)
- `TBag1` — базовый (без `OnCreate`), создаётся с нуля.
- `TBag2`–`TBag6` — с `OnCreate = MTContainerUpgrade.OnCreate`.
- Каждый рецепт использует `mode:destroy` для старой сумки — она удаляется при крафте.
- Инструменты (`Scissors`/`SharpKnife`, `SewingNeedle`) — `mode:keep` + `MayDegradeLight`.

### 1.2. `MTContainerUpgrade.lua` (центральная логика)
- `UPGRADES` — таблица `old → new` для всех категорий (TBag, TMedbag, TBelt, TWHolsterL/R, TAmmobag, TMagbag, TCanteen).
- `Pending` — очередь обработки, у каждого элемента `wait = 5`, `ticks = 60`.
- `OnCreate` — только сбор данных + снятие старого (см. §3).
- `OnTick` — перенос + надевание (см. §3).

### 1.3. Ключевые решения
- **`OnCreate` без мутаций контейнера** — иначе `getItemContainer()` вернёт `nil`, краш или потеря вещей. См. `MT_02_RULES` §1.4.
- **`ISWearClothing`** для надевания — иначе нет анимации, MP-синхронизация может сломаться. См. `MT_02_RULES` §1.1.
- **`AddItem` без явного `Remove`** — `AddItem` атомарен. См. `MT_02_RULES` §1.2.
- **`copyFluidsFrom`** для жидкостей — `Empty + addFluid` теряет смеси. См. `MT_02_RULES` §1.5.
- **Хотбар через `attachItem`** — для фляг/раций. См. `MT_02_RULES` §1.3.

## 2. Рецепты: полный список

### 2.1. TBag-цепочка (пример)
```
TBag1 → TBag2 → TBag3 → TBag4 → TBag5 → TBag6
```
Каждый шаг — `craftRecipe TBagN` с `OnCreate = MTContainerUpgrade.OnCreate`, `inputs` содержат `TBagN-1` (mode:destroy) + ресурсы.

**TBag1** — 5×Thread, 1×Twine, 10×LeatherStrips.  
**TBag2** — TBag1 + 20×Paperclip + 1×Glue + 10×Thread + 2×Twine + 10×LeatherStrips.  
**TBag3** — TBag2 + 1×Lighter + 2×Rope + 1×Glue + 15×Thread + 2×Twine + 10×LeatherStrips.  
**TBag4** — TBag3 + 50×Paperclip + 1×DuctTape + 1×Rope + 1×Glue + 20×Thread + 2×Twine + 10×LeatherStrips.  
**TBag5** — TBag4 + 1×Toolbox (любой из 6 типов) + 1×Lighter + 2×Rope + 1×HandTorch + 1×Glue + 25×Thread + 2×Twine + 10×LeatherStrips.  
**TBag6** — TBag5 + 1×WalkieTalkie (1–5) + 2×Rope + 1×HandTorch + 1×Glue + 30×Thread + 2×Twine + 10×LeatherStrips.

### 2.2. Другие цепочки
- `TMedbag1 → … → TMedbag6`
- `TBelt1 → … → TBelt6`
- `TWHolsterL1 → … → TWHolsterL6`
- `TWHolsterR1 → … → TWHolsterR6`
- `TAmmobag1_1 → … → TAmmobag1_6`
- `TAmmobag2_1 → … → TAmmobag2_6`
- `TMagbag1_1 → … → TMagbag1_6`
- `TMagbag2_1 → … → TMagbag2_6`
- `TCanteen1 → TCanteen2 → TCanteen3` — **фляги** (хотбар, не worn).

## 3. Алгоритм `MTContainerUpgrade.lua`

### 3.1. `OnCreate(craftRecipeData, character)` — только сбор
1. `getAllCreatedItems():get(0)` → `newItem`.
2. Найти `oldType` в `UPGRADES` по `newType`.
3. В `getAllConsumedItems()` найти `oldItem` по `getFullType()`.
4. Проверить: надет ли `oldItem` на персонажа (`getWornItems()`). Если да — сохранить `wornLoc = worn:getLocation()`, вызвать `character:removeWornItem(oldItem)`.
5. Если не надет — проверить хотбар: `oldItem:getAttachedSlot()`. Если `idx > -1` — сохранить `hotbarIndex`, снять через `hb:removeItem(oldItem, false)` / `character:removeAttachedItem(oldItem)`.
6. `table.insert(Pending, {newItem, oldItem, wasWorn, wornLoc, hotbarIndex, character, wait=5, ticks=60})`.

### 3.2. `OnTick` — через 5 тиков
1. `wait > 0` → `wait--`, пропустить.
2. Иначе:
   - **Перенос вещей** (если newItem — контейнер): `newContainer:AddItem(item)` для каждого item из old. **Без явного `Remove`** — `AddItem` атомарен.
   - **Перенос жидкостей:** `newFluid:copyFluidsFrom(oldFluid)` (fallback: `Empty + addFluid`).
   - **Надевание** (если `wasWorn`): `ISTimedActionQueue.add(ISWearClothing:new(char, item))`. Fallback: `setWornItem` + `resetModel` + `triggerEvent`.
   - **Хотбар** (если `hotbarIndex`): `hb:attachItem(item, slot.def.attachments[item:getAttachmentType()], idx, slot.def, false)` + `syncItemFields`.
   - `table.remove(Pending, i)`.
3. Если `newContainer` ещё `nil` — `ticks--`, при `ticks <= 0` — `table.remove` (страховка от вечного ожидания).

### 3.3. Cleanup старого предмета
Если крафт по какой-то причине не удалил `oldItem` (нет `mode:destroy`, десинк) — снять с рук, с хотбара, удалить из контейнера + `sendRemoveItemFromContainer` в MP. **Не терять!**

## 4. Специфичные грабли

- **`OnCreate` вызывается до полной инициализации** `newItem`. `getItemContainer()` = `nil`. Отсюда `wait = 5` (см. `MT_02_RULES` §1.4).
- **Фляга vs сумка:** `IsInventoryContainer()` = `true` только для контейнеров. Для фляг — только `attachItem` + `copyFluidsFrom`.
- **`getAttachedSlot()` возвращает `-1`** для не прикреплённых. Проверять `> -1`.
- **Двойные запятые в рецептах** (патроны/магазины) — исправлены.

## 5. TODO
- [ ] Финальный тест обеих версий в игре (SP): крафт надетой сумки с вещами + с водой во фляге.
- [ ] Тест на переполненном контейнере: убедиться, что `AddItem = nil` → вещи остаются в старой сумке (не теряются).
- [ ] Тест в MP: синхронизация (`sendRemoveItemFromContainer`, `syncItemFields`).