# MT_MODS/MT_Zeds/DATABASE.md

# [MT] Zeds — DATABASE

> **[MT] AI reminder:** не гадай API — запрашивай декомпил (`MT_04_NAVIGATION` §1). Ищи идиому до примитива. Помечай `[?]` предположения. Полный протокол — `MT_01_AI_PROTOCOL`.
> **Общие правила:** `MT_02_RULES`. **Паттерн патча:** `MT_03_KNOWLEDGE` §3.4.

## 1. Архитектура

**Файлы:**
- `Patch_ZombieIdleState.java` — основной.
- `Patch_ZombieGroupManager.java` — опциональный (провал, см. §2).

**Стек:** Java-патчи (ZombieBuddy).

## 2. Провалы, ограничения и непроверенные варианты

### 2.1. `ZombieGroupManager.shouldBeInGroup` через `isAdvice=false` — ПРОВАЛ
**Что пробовали:**
```java
@Patch(className = "zombie.ai.ZombieGroupManager", methodName = "shouldBeInGroup", isAdvice = false)
public class Patch_ZombieGroupManager {
    public static boolean shouldBeInGroup(ZombieGroupManager manager, IsoZombie zombie) {
        return false;
    }
}
```
**Результат:** `regrouped` остался 58%. Патч применяется (`Found patch class`), но метод не заменяется.

### 2.2. `ZombieGroupManager.update` через `isAdvice=false` — ПРОВАЛ
**Что пробовали:** полная замена `update(IsoZombie)` пустым методом.
**Результат:** `regrouped` 76%. `MethodDelegation` не работает в текущей версии ZombieBuddy для этих методов.

### 2.3. `@Patch.Replace` отсутствует
См. `MT_03_KNOWLEDGE` §2.2.

### 2.4. Вариант B — runtime-флаг `Core` — ПРОТЕСТИРОВАТЬ КАК ВОЗМОЖНЫЙ ВЫХОД

**Это не зафиксированный провал**, а альтернативный путь, который ещё не проверялся в игре.

`ZombieGroupManager.shouldBeInGroup()` проверяет флаг `Core`:

```java
if (!Core.getInstance().isZombieGroupSound()) {
    return false;
}
```
`Java_Decompile/zombie/ai/ZombieGroupManager.java:69`

`Core` добавлен в Lua-exposer (`Java_Decompile/zombie/Lua/LuaManager.java:1862`), `getCore()` — глобальная Lua-функция (`LuaManager.java:6287–6288`). Возможный runtime-выключатель:

```lua
getCore():setZombieGroupSound(false)
print(getCore():isZombieGroupSound()) -- ожидается false
```

Флаг по умолчанию `true` (`Core.java:331`), единственная ванильная запись `true` — в `IngameState.exit` (`IngameState.java:1014`). Поэтому выставлять его нужно при `OnGameStart`; после выхода из игры/нового входа флаг может сброситься.

**Ожидание:** существующие группы распадутся через `ZombieGroup.update()` (`ZombieGroup.java:72`), новые создаваться не будут. Сам менеджер продолжит тикать, но группового AI и rally-поведения не будет.

**Что проверить:**
- [ ] метод `setZombieGroupSound` реально доступен из Lua (Kahlua);
- [ ] `regrouped` падает до ~0;
- [ ] нет побочных эффектов на звук/остальной AI;
- [ ] флаг не сбрасывается при новой игре/перезагрузке.

`[?]` Не тестировано. Если Lua-метод недоступен — fallback на Java-патч `shouldBeInGroup` через advice, **не** через `isAdvice=false`.

## 3. Что работает (смягчение)

### 3.1. Очистка `z.group` в Idle
**Где:** `Patch_ZombieIdleState.java`, `onExit`.
**Что делает:** удаляет зомби из ванильной группы в момент «думания» (раз в ~1 с при `THINK_ODDS = 60`).

```java
ZombieGroup g = z.group;
if (g != null) {
    g.remove(z); // внутри remove() z.group = null
}
```

**Почему помогает:** `ZombieGroupManager` тянет к лидеру только `Idle`-участников в окне `tickCount == 0`. Зомби, которые почти всё время идут, в него попадают редко.

**Чего НЕ делает:** не блокирует менеджер навсегда. В следующем окне `update()` снова найдёт группу в `rallyTravelDistance` или создаст новую.

**Честная диагностика:** счётчик `regrouped` инкрементится **до** `remove()`, если `z.group != null`. `inGroup` **после** `remove()` всегда 0 — не доказательство.

**Результат:** `regrouped` 58–76%. Менеджер очень активен.

**Не убирать:** без чистки `THINK_ODDS = 60` вообще не спасёт (зомби в `Idle` → менеджер стянет сразу).

### 3.2. `THINK_ODDS = 60`
- 98% вызовов отсеиваются.
- `pathToLocation(tx, ty, tz)` + `allowRepathDelay = 200.0F`.
- `MAX_TRAVEL = 50`, `MAX_PLAYER_DIST = 150`.

### 3.3. Совместимость с `PZ_Optimization`
- Счётчики — `LongAdder`.
- Снимок игроков — `volatile float[]` с атомарной подменой ссылки.
- `isRemoteZombie()` закомментирован (см. `MT_03_KNOWLEDGE` §2.3).
- `ZombieGroup g = z.group;` перед `remove()` — избегает NPE.
- `DEBUG = false` для релиза.

## 4. Диагностика и тесты

### 4.1. Тест «менеджер тянет followers»
Менеджер тянет, если: MP **или** `getHoursSurvived() >= 2` **или** `Core.debug`. Для лидеров — без `Core.debug`.

### 4.2. Публичное поле `group` у `IsoZombie`
Диагностика: `z.group` — есть ли группа.

### 4.3. Счётчики
- `passedAll` — сколько зомби прошло через наш OnExit.
- `regrouped` — сколько были в группе **до** `remove()`.
- `inGroup` — сколько в группе **после** `remove()` (всегда 0 — не доказательство).

## 5. Специфичные грабли

- **`isRemoteZombie()`** ломается с `PZ_Optimization` (см. `MT_03_KNOWLEDGE` §2.3).
- **Race condition** на статических `int++` и перезаписи массивов игроков (см. §3.3).
- **`ZombieGroupManager` не отключается** через `isAdvice=false` (см. §2). **Возможный выход:** runtime-флаг `Core.setZombieGroupSound(false)` — протестировать (см. §2.4).
- **Гейт `stateEventDelayTimer > 0`** в `ZombieIdleState.execute` — ванильный `execute` сбрасывает таймер **до** нашего `OnExit`.
- **`ZombieGroupManager.update()`** в конце `IsoZombie.update()` (~3506) — если зомби в `Idle`, менеджер тянет к лидеру.

## 6. Открытые вопросы
- [?] Как отключить ванильный `ZombieGroupManager` без `isAdvice=false`? (в приоритете)
  - [?] Протестировать `getCore():setZombieGroupSound(false)` как возможный выход (см. §2.4).
- [?] Свой Merge параллельно с ванильным — не будет ли конфликта?
- [?] Как синхронизировать `WZ_Role` в MP без лагов?