# MT_03_KNOWLEDGE.md

> **Растущая база знаний проекта [MT].** Три подраздела: симптомы → где искать канон, подтверждённые факты API, проверенные паттерны.
> Правила движка и соглашения — в `MT_02_RULES`. Протокол работы — в `MT_01_AI_PROTOCOL`.
> Build 42.21. Декомпил: `MT_04_NAVIGATION` §1. **История:** `MT_DOCS/CHANGELOG.md`.
>
> **Принцип:** этот файл не дублирует `MT_02`. Если факт стал правилом — он живёт в `MT_02`, а здесь только ссылка. Если факт — паттерн или подтверждение API — он живёт здесь.

## 1. Симптомы → канон

> Быстрый поиск по симптому. Если симптом встречается впервые — добавить строку сюда и завести правило/факт в каноне.

### 1.1. Крафт и предметы

| Симптом | Куда |
|---|---|
| Сумка пропала после крафта, вещи внутри исчезли | `MT_02_RULES` §1.4 + `MT_MODS/MT_TacticalBags/DATABASE` |
| `ISWearClothing` не срабатывает после крафта | `MT_02_RULES` §1.1 + §1.3 |
| `AddItem` вернул `nil`, предмет не добавлен | `MT_02_RULES` §1.2 |
| Явный `Remove` перед `AddItem` → потеря предмета | `MT_02_RULES` §1.2 (запрещено) |
| Вода/смесь пропала при переносе из старой фляги | `MT_02_RULES` §1.5 |
| Фляга не прикрепилась на хотбар | `MT_02_RULES` §1.3 |
| Слот worn-предмета пустой, хотя вещь надета | `MT_02_RULES` §1.3 (`WornItem:getLocation()`) |

### 1.2. UI и графика

| Симптом | Куда |
|---|---|
| Кольцо `DrawTextureAngle` не фейдится | `MT_02_RULES` §1.7 (alpha в 8-м аргументе) |
| Текст маленького размера не отрисовывается | `MT_02_RULES` §1.7 (API нет в B42.21) |
| Кнопка `setX(-500)` уехала за экран и не кликается | `MT_02_RULES` §1.7 (`keepOnScreen = false`) |
| Панель «дрожит» при Drag&Drop | `MT_02_RULES` §1.7 (экранные координаты) |
| UI-классы крашат при старте | `MT_02_RULES` §1.7 (`OnGameBoot`) |
| Батарея `nil` при вызове `Battery:getCurrent(...)` | §2.1 (функция, не метод) |
| `panelHidden=true` + `linePos="top"` → Unhide вне bbox | `MT_MODS/MT_SmartWatch/DATABASE` §4.2 |
| Иконки Hide/Lock визуально не выровнены при `scale≠1.0` | `MT_MODS/MT_SmartWatch/DATABASE` §4.1 |

### 1.3. Анимации и оружие

| Симптом | Куда |
|---|---|
| Кастомная стойка не проигрывается в Idle | `MT_02_RULES` §1.10 (`maskingright` + приоритет) |
| XML загружен, маска применяется, но играется ванильная стойка | `MT_02_RULES` §1.10 (имя Action в FBX) |
| Кисть выворачивается при кастомной анимации | `MT_02_RULES` §1.10 (2 кости пальцев, симметрия) |
| Стойка «слетает» при ходьбе | `MT_02_RULES` §1.10 (`isCurrentActionPathfinding`) |

### 1.4. Зомби и AI

| Симптом | Куда |
|---|---|
| Зомби стягиваются в кучи после патча Idle | `MT_MODS/MT_Zeds/DATABASE` §3.1 |
| `regrouped` 58–76% — менеджер возвращает в группы | `MT_MODS/MT_Zeds/DATABASE` §2 + §3.1 |
| `isRemoteZombie()` отсеивает всех в SP | `MT_02_RULES` §1.6 + `MT_MODS/MT_Zeds` §3.3 |
| Путь сбрасывается на следующем кадре | `MT_MODS/MT_Zeds/DATABASE` §5 |

### 1.5. Java-патчи (ZombieBuddy)

| Симптом | Куда |
|---|---|
| `IllegalAccessError: tried to access private` | `MT_02_RULES` §2.5 (`public`) |
| `cannot find symbol: class Replace` | §2.2 (`@Patch.Replace` отсутствует) |
| `isAdvice=false` не заменяет метод | §2.2 (ограничение ZombieBuddy) |
| Race condition в `PZ_Optimization` | `MT_MODS/MT_Zeds/DATABASE` §3.3 |

### 1.6. Файлы и загрузка

| Симптом | Куда |
|---|---|
| `_Data.lua` не загрузился раньше основного файла | `MT_02_RULES` §1.8 + §2.5 (ASCII) |
| `getFileWriter` отказывает в записи | `MT_02_RULES` §1.8 (allowlist) |
| Кириллица в консоли — кракозябры | `MT_02_RULES` §2.1 (только ASCII) |

---

## 2. Подтверждённые факты API (декомпил 42.21)

> Факты, проверенные по декомпилу с указанием `file:line`. Не вошли в `MT_02` как правило (специфичны для мода или узкие), но подтверждены.

### 2.1. `Battery.getCurrent` — функция, не метод

**Факт:** в Lua-биндингах `Battery.getCurrent(watch)` вызывается **только через точку**. Вызов через двоеточие (`Battery:getCurrent(watch)`) передаёт `Battery` как первый аргумент, что даёт `nil` или ошибку.

**Источник:** `MT_MODS/MT_SmartWatch/DATABASE` §1 + `Battery.lua` в `Lua_Decompile/`.

**Применение:** во всех модах, где есть объекты с похожим Lua-биндингом.

### 2.2. `@Patch.Replace` отсутствует в текущей версии ZombieBuddy

**Факт:** в ZombieBuddy на момент B42.21 нет аннотации `@Patch.Replace`. Попытка использования → `cannot find symbol: class Replace`.

**Что работает:** `@Patch.OnEnter` + `@Patch.OnExit` (advice-стиль).
**Что НЕ работает:** `isAdvice = false` для полной замены метода — тесты на `ZombieGroupManager.shouldBeInGroup` и `update` показали, что оригинал всё равно выполняется (regrouped 58–76%).

**Применение:** при написании Java-патчей не пытаться полностью заменить метод — только обернуть.

### 2.3. `isRemoteZombie()` ломается с `PZ_Optimization`

**Факт:** `PZ_Optimization` выносит обновление зомби в worker-потоки, помечая их «remote». В SP-игре почти все зомби отсеиваются по `remote=...`.

**Что делать:** закомментировать `isRemoteZombie()` в Java-патче, либо проверять через `isClient()` / `isServer()`.

**Применение:** все Java-патчи, работающие с зомби.

### 2.4. `getFileWriter` allowlist — подтверждено

**Факт:** PZ разрешает запись только для `.ini`, `.cfg`, `.txt`, `.log`. Другие расширения → ошибка записи.

**Источник:** `MT_MODS/MT_SmartWatch/DATABASE` §3.6 + `MT_02_RULES` §1.8.

**Применение:** любая запись конфигов мода — только эти расширения.

### 2.5. `ISWalkToTimedAction` не существует в B42

**Факт:** в B42 ходьба реализована через `ISPathFindAction`. Класса `ISWalkToTimedAction` нет.

**Что использовать для проверки ходьбы:** `player:isCurrentActionPathfinding()` (нативный метод).

**Источник:** `MT_02_RULES` §1.10 (проверено на `MT_Weapons`).

**Применение:** `MT_Weapons` — не считать ходьбу «busy» и не спамить `transmitModData`.

### 2.6. `RightHandMask` работает для мили-оружия

**Факт:** ранее считалось, что `RightHandMask` не работает для `HandWeapon` в `Idle`. Это **опровергнуто**. Работает при:
1. Своих XML-узлах в `media/animsets/player/maskingright/`.
2. Установке переменной из Lua: `player:setVariable("RightHandMask", "MyIdleMask")`.
3. `m_ConditionPriority > 0` в XML (перебивает ванильные узлы).

**Источник:** `MT_MODS/MT_Weapons/DATABASE` §9 + `MT_02_RULES` §1.10.

**Применение:** `MT_Weapons` — все кастомные стойки через подслой `maskingright`.

### 2.7. `HeadCondition` работает нативно для мили-оружия

**Факт:** в B42 `HeadCondition` (разделение прочности лезвия/ручки) работает для мили-оружия нативно, как в ванильных топорах. `HeadConditionLowerChanceMultiplier` тоже работает.

**Применение:** `MT_Weapons` — не изобретать свою систему прочности, использовать нативную.

### 2.8. `PZAPI.ModOptions` — ComboBox индексы

**Факт:** `cb:getValue()` возвращает **индекс** (1, 2, 3...), не текст. Заполнение: `local cb = options:addComboBox(...)`, затем `cb:addItem(name, isDefault)` в цикле. Разделители — `options:addTitle(text)`; `options:addSeparator()` рисует линию.

**Применение:** все `ModOptions`-меню (`MT_Weapons`, `MT_SmartWatch`).

---

## 3. Паттерны (проверенные рецепты)

> Не факт и не соглашение — **рекомендация**. Проверено в бою. Если применяешь — ссылайся на этот раздел.

### 3.1. Безопасный крафт с переносом содержимого (TBag-паттерн)

**Задача:** при крафте заменить предмет A на B, перенести содержимое и жидкости, надеть B, сохранить MP-синхронизацию.

**Паттерн:**
1. **`OnCreate`** — только сбор данных:
   - `getAllCreatedItems():get(0)` → newItem
   - `getAllConsumedItems()` → ищем oldItem по `getFullType()`
   - Проверяем, был ли oldItem надет / на хотбаре → сохраняем `wasWorn` / `wornLoc` / `hotbarIndex`
   - Снимаем oldItem (если надет) **без** мутации контейнера
   - Кладём всё в `Pending`-таблицу с `wait = 5` и `ticks = 60`
2. **`OnTick`** — через 5 тиков (когда движок закончил):
   - Если newItem — контейнер (`IsInventoryContainer()`): перенос вещей через `newContainer:AddItem(item)` (без явного `Remove` — `AddItem` атомарен)
   - Перенос жидкостей через `newFluid:copyFluidsFrom(oldFluid)` (fallback: `Empty + addFluid`)
   - Надевание через `ISTimedActionQueue.add(ISWearClothing:new(char, item))`
   - Хотбар через `getPlayerHotbar():attachItem(item, slot, idx, def, false)`
   - `syncItemFields(char, item)` если существует
3. **Cleanup** — на каждом тике `data.ticks = data.ticks - 1`; при `ticks <= 0` — `table.remove(Pending, i)` (страховка от вечных попыток).

**Источник:** `MT_TacticalBags` (`MTContainerUpgrade.lua`), проверено в бою.

**Анти-паттерн:** перенос содержимого внутри `OnCreate` → `getItemContainer()` возвращает `nil` → краш или потеря вещей.

### 3.2. Надевание предмета с анимацией

**Задача:** надеть предмет на персонажа так, чтобы была анимация, MP-синхронизация и обновление UI.

**Паттерн:**
```lua
if ISWearClothing and ISTimedActionQueue then
    ISTimedActionQueue.add(ISWearClothing:new(char, item))
else
    -- fallback: прямой setWornItem + resetModel + triggerEvent
    char:setWornItem(loc, item)
    char:resetModel()
    triggerEvent("OnClothingUpdated", char)
end
```

**Anti-pattern:** `char:setWornItem(loc, item)` без `ISWearClothing` в основном пути — работает в SP, но не даёт анимации и может конфликтовать в MP.

### 3.3. Прикрепление на хотбар

**Задача:** прикрепить предмет (фляга, рация, фонарь) на хотбар.

**Паттерн:**
```lua
local hb = getPlayerHotbar and getPlayerHotbar(char:getPlayerNum())
if not hb then return end  -- сервер или нет хотбара
local slot = hb.availableSlot[idx]
if not slot or not hb:canBeAttached(slot, item) then return end
hb:attachItem(item, slot.def.attachments[item:getAttachmentType()], idx, slot.def, false)
if syncItemFields then syncItemFields(char, item) end
if ISInventoryPage then ISInventoryPage.renderDirty = true end
```

**Anti-pattern:** прямой `char:setAttachedItem(...)` без проверки `canBeAttached` — тихий провал.

### 3.4. Java-патч Idle-состояния зомби (Zeds-паттерн)

**Задача:** заставить зомби строить длинные пути, не стягиваться в группы.

**Паттерн:**
```java
@Patch(className = "zombie.ai.states.ZombieIdleState", methodName = "execute")
public class Patch_ZombieIdleState {
    @Patch.OnExit
    public static void onExit(ZombieIdleState state, IsoZombie z) {
        if (z == null) return;
        if (Rand.Next(100) > THINK_ODDS) return;  // THINK_ODDS = 60, 98% отсеивается
        // pathToLocation(tx, ty, tz) + allowRepathDelay = 200.0F
        // Очистка z.group через g.remove(z) — смягчение менеджера
    }
}
```

**Ключевые моменты:**
- `THINK_ODDS = 60` — экономия CPU (98% вызовов отсеиваются).
- `allowRepathDelay = 200.0F` — блокирует `pathToCharacter`/`pathToLocationF`, но **не защищает** от `ZombieGroupManager`.
- Очистка `z.group` — **смягчение**, не решение. `regrouped` остаётся 58–76%.
- `isRemoteZombie()` закомментирован (см. §2.3).
- Для `PZ_Optimization` — `LongAdder` для счётчиков, `volatile float[]` для снимка игроков.

**Источник:** `MT_Zeds` (`Patch_ZombieIdleState.java`).

**Anti-pattern:** `@Patch.Replace` — отсутствует в ZombieBuddy (см. §2.2).

### 3.5. Кастомная стойка для мили-оружия

**Задача:** кастомная стойка (Idle/Walk/Run/Aim) для мили-оружия в B42.

**Паттерн:**
1. **XML-узлы** в `media/animsets/player/maskingright/`, `m_ConditionPriority > 0`.
2. **Из Lua** — `player:setVariable("RightHandMask", "MyIdleMask")` при экипировке / смене оружия.
3. **Watchdog** на 5 тиков после выхода из `busy` — принудительная запись маски (перебивает баг двуручки в одной руке).
4. **Проверка ходьбы** — `player:isCurrentActionPathfinding()`, не считать ходьбу «busy».

**Anti-pattern:** полагаться на `primaryAnimMask` из скрипта предмета — игнорируется движком для `HandWeapon`.

**Источник:** `MT_Weapons` (`MT_MeleeStances_01_Data.lua` + `_03_Core.lua`).

### 3.6. Чтение репозитория мода по URL

**Задача:** прочитать файл из репозитория мода, где путь содержит `[MT]` (скобки + пробел).

**Паттерн:** URL-encode спецсимволы:
- `[` → `%5B`
- `]` → `%5D`
- пробел → `%20`

**Альтернативы:** web UI сам кодирует; `git clone` + локальное чтение; GitHub API `repos/{owner}/{repo}/git/trees/main?recursive=1` (возвращает пути в исходном виде).

**Источник:** `MT_MODS/MT_SmartWatch/DATABASE` §3.6.

---

## 4. Что делать при новой находке

1. Если это **правило** (проверяемо по декомпилу / соглашение) — в `MT_02_RULES` §1 или §2.
2. Если это **симптом** — добавить строку в §1 этого файла (таблица «симптом → канон»).
3. Если это **факт API** с `file:line` — в §2 этого файла.
4. Если это **проверенный рецепт** — в §3 этого файла.
5. Если это **специфика одного мода** — в `MT_MODS/<имя>/DATABASE`.
6. При правке — запись в `MT_DOCS/CHANGELOG.md`. Только для вех (новый факт API, новый паттерн, симптом, меняющий канон). Мелкие правки — git log.