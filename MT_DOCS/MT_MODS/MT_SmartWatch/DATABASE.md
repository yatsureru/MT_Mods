# MT_MODS/MT_SmartWatch/DATABASE.md

# [MT] SmartWatch — DATABASE

> **[MT] AI reminder:** не гадай API — запрашивай декомпил (`MT_04_NAVIGATION` §1). Ищи идиому до примитива. Помечай `[?]` предположения. Полный протокол — `MT_01_AI_PROTOCOL`.
> **Общие правила:** `MT_02_RULES`. **Симптомы и паттерны:** `MT_03_KNOWLEDGE`.

## 1. Контекст

**Мод:** `[MT] SmartWatchFramework` (B42.21)
**Задача HUD:** иконка при надевании часов; при наличии ядра — кольцо, батарейка, %; ПКМ → настройки. При наличии ядра — Тактический хотбар.

**Связанные модули в `shared/`:** `Watch.lua`, `OSCore.lua`, `ChipSystem.lua`, `Battery.lua`, `BatteryDrain.lua`, `TacticalChipRegister.lua`.

## 2. Структура HUD

```
[MT] SmartWatchFramework/42/media/
├── lua/client/MT_SmartWatchFramework/HUD/
│   ├── HUD_Init.lua              — entry point, синглтоны, троттлинг 8 кадров
│   ├── HUD_State.lua             — State + Settings, чтение/запись .ini
│   ├── HUD_Base.lua              — ISUIElement (fade, drag, pin, hover)
│   ├── HUD_Icon.lua              — иконка + кольцо + Battery.render
│   ├── HUD_Battery.lua           — 5 тиров, кэш, пульс <5%
│   ├── HUD_SettingsPanel_WatchOnScreen.lua
│   ├── HUD_TacticalSlots.lua     — слоты, Drag&Drop, кулдауны, Hide/UnHide, полоса, замок
│   ├── HUD_SettingsPanel_Hotbar.lua
│   ├── HUD_Interface.lua
│   ├── HUD_Interface_TacticalChips.lua
│   └── HUD_ModOptions.lua.disabled
└── UI/
    ├── OnScreenSmartWatchHUD/*.png
    └── OnScreenTacticalPanelHUD/*.png
```

## 3. Архитектурные правила

### 3.1. Namespace
- `MT_SmartWatch.HUD.State` / `.Settings` — данные
- `MT_SmartWatch.HUD.Base / Icon / SettingsPanel_WatchOnScreen / TacticalSlots` — модули
- `MT_SmartWatch.HUD.icon` / `.tacticalSlots` — синглтоны UI
- `MT_SmartWatch.HUD.dragPreview` — превью Drag&Drop

### 3.2. Имена классов
`:derive("MT_SmartWatch_HUD_XXX")`.

### 3.3. Лог-префикс
`[MT Smart Watch - HUD]` для grep. Только ASCII в `print()` (см. `MT_02_RULES` §2.1).

### 3.4. Хранилище
- **Единственный источник правды:** `Zomboid/MT_SmartWatch_HUD.ini`
- **Runtime** — `State.update(player)` каждые 8 кадров.

### 3.5. Переводы
- Ключи `UI_MTSW_HUD_*` в `Translate/{EN,RU}/ui.json`. Trailing comma разрешена.

### 3.6. Чтение репозитория (URL-encoding)
Пути содержат `[MT]` → encode: `[`→`%5B`, `]`→`%5D`, пробел→`%20`.
Пример: `https://raw.githubusercontent.com/yatsureru/-MT-SmartWatch/main/%5BMT%5D%20SmartWatchFramework/42/...`
Альтернативы: web UI, git clone, GitHub API trees.

### 3.7. Формат вывода полного документа
Обёртка **4 бэктика** (` ````markdown ... ```` `). Внутри есть ```lua/```ini — тройные разорвут.

### 3.8. Текстуры обязательны, fallback убран
Для линий, замка, Hide/UnHide, пустого слота и пустой батарейки — **fallback `drawRect` удалён**. Если PNG нет — элемент не рисуется (лучше, чем серый прямоугольник, маскирующий баг).
Исключения (fallback оставлен): `drawClockSweep` (программная отрисовка КД), tooltip (программный `drawRect` фона).

## 4. Специфичные грабли (только SmartWatch)

### 4.1. 🚨 Визуальный padding в `HidePanel.png` — постоянная компенсация (НЕ УБИРАТЬ)

У `HidePanel.png` / `LockClosed.png` прозрачные поля несимметричны. Выровнять по невидимым полям **физически невозможно** — `drawTextureScaled` рендерит по bounding box, не по видимому контенту. При `scale ≠ 1.0` отступ Hide-Слот меньше отступа Слот-Замок.

```lua
local HIDE_SLOTS_BONUS_PER_SCALE = 6
-- бонус = (scale - 1.0) * 6
-- scale 1.0 → 0, scale 1.5 → +3px
```

Только в `getSlotsOriginX()`. `getLockOffsetX()` не компенсируется — там нет проблемы.

### 4.2. 🚨 `panelHidden=true` + `linePos="top"` → Unhide вне bbox

При `panelHidden` `setHeight(outer)` — кнопка уходит ниже `outer`, клики не регистрируются.

**Фикс:**
```lua
if self.panelHidden then
    self:setWidth(hideW)
    self:setHeight(outer + self:getSlotsOffsetY())
    return
end
```

### 4.3. Другие грабли (общие — см. ссылки)
- `DrawTextureAngle` alpha в 8-м аргументе — `MT_02_RULES` §1.7
- Масштабирование текста — нет API — `MT_02_RULES` §1.7
- `Battery.getCurrent(watch)` через точку — `MT_03_KNOWLEDGE` §2.1
- `getTimestampMs()` вместо `getTimeStep()` — `MT_02_RULES` §1.7
- Drag&Drop: 4 метода + `setCapture` — `MT_02_RULES` §1.7

## 5. Настройки (.ini)

`Zomboid/MT_SmartWatch_HUD.ini`

| Ключ | Значения | По умолчанию |
|---|---|---|
| `VERSION` | 10 | — |
| `iconSize` | 256 / 128 / 64 | 128 |
| `fadeTime` | 0..6 | 0 |
| `circleSpeed` | 1..4 | 3 |
| `circleDirection` | 1..2 | 2 |
| `tacticalLock` | true/false | false |
| `tacticalScale` | 0.5 / 0.75 / 1.0 / 1.25 / 1.5 | 1.0 |
| `tacticalGap` | 0 / 2.5 / 5 / 7.5 / 10 | 2.5 |
| `tacticalFadeTime` | 0..6 | 0 |
| `tacticalLinePosition` | `bottom` / `top` / `hide` | `bottom` |
| `tacticalPanelHidden` | true/false | false |
| `tacticalHideExtraButtons` | true/false | false |

**Санити:** если `tacticalHideExtraButtons = true` → `tacticalPanelHidden` форсится в `false`. При несовпадении `VERSION` → полный сброс layout-настроек.

## 6. Поток событий

```
OnGameBoot → buildBaseClass / buildPanelClass / buildClass
OnGameStart → State.load()
OnPlayerUpdate (каждые 8 кадров) → State.update(player) → ensureIcon / ensureTacticalSlots
TacticalSlots:render() каждый кадр:
  → Кнопка Hide/UnHide (если !hideExtra)
  → Если panelHidden → return
  → Полоса (если linePos ≠ "hide")
  → Слоты + чипы (с КД / пустой батарейкой)
  → Замок (если !hideExtra)
  → Tooltip
ЛКМ по Hide/UnHide → togglePanel() → save .ini → recalcSize()
ПКМ по хотбару → openHotbarSettingsPanel → State.save() → .ini
```

## 7. Debug & Console

### 7.1. Дебаг-клавиши
| Клавиша | Функция |
|---|---|
| **1** | `Debug.testInstallLogicCore()` → `Base.MTSW_Core_Logic` |
| **2** | `Debug.testRemoveOSCore()` — снять ядро |
| **3** | `Debug.testInstallTacticalCore()` → `Base.MTSW_Core_Tactical` |

Ядро `Base.MTSW_Core_042` штатной клавишей не ставится.

### 7.2. Установка ядра 042 через консоль
```lua
local player = getPlayer()
local watch = MT_SmartWatch.Watch.getEquippedWatch(player)
local core = player:getInventory():getFirstType("Base.MTSW_Core_042")
if watch and core then
    local ok = MT_SmartWatch.OSCore.installCore(watch, core)
    print("[MT SW] install = " .. tostring(ok))
end
```

### 7.3. Через ПКМ
ПКМ по надетым часам → «Установить ядро». Ядро расходуется из инвентаря.