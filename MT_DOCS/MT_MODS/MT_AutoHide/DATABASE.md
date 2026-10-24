# MT_MODS/MT_AutoHide/DATABASE.md

# [MT] AutoHide — DATABASE

> **[MT] AI reminder:** не гадай API — запрашивай декомпил (`MT_04_NAVIGATION` §1). Ищи идиому до примитива. Помечай `[?]` предположения. Полный протокол — `MT_01_AI_PROTOCOL`.
> **Общие правила:** `MT_02_RULES`. **Паттерны UI:** `MT_03_KNOWLEDGE` §3.

## 1. Архитектура

**Файл:** `MT_UI_Hide_NavBar.lua`.

### 1.1. Плавное скрытие
- `getTimestampMs()` (не `getTimeStep()` — нет в B42.21).
- `currentVisualX + (targetX - currentVisualX) * ease` (Smoothstep).
- Разные кривые для заезда/уезда.
- `panel:setX(currentVisualX)` каждый кадр.

### 1.2. Настройки (план)
- Радиус «рамки» — пиксели.
- Время заезда/уезда — секунды.
- Одна кривая Smoothstep для обеих сторон (упрощение).

## 2. Специфичные грабли

### 2.1. Кнопки «Поднять/Опустить» не ловятся
**Симптом:** кнопки не поймать через `UIManager` или `getPlayerContextMenu`.
**Причина:** создаются в `ISMoveableCursor` / `ISMoveableInfoWindow`, не регистрируются как отдельное окно.
**Что проверено:** `ISMoveableCursor`, `ISMoveableInfoWindow.infoPanels[0]`, `getCell():getDrag(0)` — не помогло.
**Статус:** `[?]` не решено. Гипотеза — фикс рамки +40.

### 2.2. Общие грабли (см. ссылки)
- `getTimeStep()` не существует — `MT_02_RULES` §1.7
- `keepOnScreen = false` перед `setX(-500)` — `MT_02_RULES` §1.7
- UI-классы через `OnGameBoot` — `MT_02_RULES` §1.7

## 3. Открытые вопросы
- [?] Где брать ссылку на левую панель? `getPlayerInventory()`? `getInventoryPage()`?
- [?] Как отловить кнопки «Поднять/Опустить» без хака?
- [?] Совместимость с другими UI-модами (включая `MT_SmartWatch`)?