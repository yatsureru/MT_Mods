# MT_MODS/MT_Weapons/DATABASE.md

# [MT] Melee Weapons — DATABASE

> **[MT] AI reminder:** не гадай API — запрашивай декомпил (`MT_04_NAVIGATION` §1). Ищи идиому до примитива. Помечай `[?]` предположения. Полный протокол — `MT_01_AI_PROTOCOL`.
> **Общие правила:** `MT_02_RULES`. **Симптомы и паттерны:** `MT_03_KNOWLEDGE`.

## 1. Архитектура и контекст

**2 модуля:**
1. `MT_Stance` — стойки. Данные (`MT_MeleeStances_Config.lua`, `MT_MeleeStances_Data.lua`) лежат в корне, логика — в `MT_Stance/Runtime/`. Рефактор выполнен; гейт по лезвию включён.
2. `MT_Weapons` — ручки, лезвия, аттачменты. Сейчас строится вертикальный срез: тестовая катана + нативный `WeaponPart`.

**Формула Power:** `D·0.38 + C·0.22 + S·0.18 + U·0.12 + Wt·0.10 + (5 − Cost)·0.15`
**Целевые Power:** T1 (3.5–4.3), T2 (4.5–5.1), T3 (5.3–5.9), T4 (5.8–6.5).

### 1.1. Текущий этап — вертикальный срез

Цель: доказать нативно, что связка «ручка + лезвие» работает до реализации всех 18 ручек и 16 лезвий.

В срез входит:
- тестовая ручка `Base.MT_Katana_Test`;
- тестовое лезвие `Base.MT_Attachment_Katana_Blade_Test` (`ItemType = base:weaponpart`);
- нативный `ModelWeaponPart` + `attachment Blade` в модели ручки;
- контекстное меню: `MTAttachWeaponPart` для установки и `ISRemoveWeaponUpgrade` для снятия;
- проверка save/load и MP-синхронизации предмета;
- проверка, что ручка без лезвия не получает стойку (после включения гейта).

Срез **не** включает: 18 ручек, 16 лезвий, текстуры/скины, гравировки, свой UI Т, урон/прочность, крафт.

## 2. Система предметов (Ручка + Лезвие)

- **4 кор-категории:** `mtmw:knife`, `mtmw:katana`, `mtmw:onehandaxe`, `mtmw:axe`.
- **18 ручек:** разные визуалы, статы одинаковые (0 урона).
- **16 лезвий:** по 4 лезвия на каждую категорию (T1–T4). Катана — 4 лезвия, совместимы со всеми ручками катан; нож — 4, со всеми ручками ножей; и так далее.
- **Один item лезвия — разные визуалы на ручках:** item-скрипт лезвия один на категорию/тир, а меш, текстура и положение для каждой ручки задаются её `ModelWeaponPart`.
- **10 текстур:** накладываются на лезвие (Эмеральд, Рубин).

### 2.1. Роль `mtmw:handle`

`mtmw:handle` — **маркер роли**, а не третье условие стойки:
- ставится на каждую ручку;
- семейный тег (`mtmw:katana`, `mtmw:knife`, ...) ставится только на ручки и всегда вместе с `mtmw:handle`;
- лезвие не получает ни `mtmw:handle`, ни семейный тег;
- отдельный `mtmw:blade` не нужен: лезвие определяется нативным `WeaponPart` с `PartType = Blade`.

Стойка проверяет **два условия**: семейный тег + `WeaponState.isComplete(item)`. Резолвер дополнительно требует `mtmw:handle`, чтобы семейный тег, случайно оставленный на не-ручке, не включил стойку.

### 2.2. Контракт `WeaponState.isComplete`

```lua
-- media/lua/shared/MT_MeleeWeapons_Common/WeaponState.lua
function WeaponState.isComplete(item)
    if not item then return false end
    if not Tags then return false end                             -- fail-closed
    if not Tags.get("mtmw:handle") then return false end           -- registry unavailable
    if not Tags.hasTag(item, "mtmw:handle") then return true end  -- не ручка: не наша ответственность
    if not Attachments then return false end                      -- fail-closed
    return Attachments.hasBlade(item)                              -- item:getWeaponPart("Blade") ~= nil
end
```

Правила:
- **Не ручка** → `true`. Ванильное/чужое оружие не блокируется.
- **Ручка + лезвие** → `true`.
- **Ручка без лезвия** → `false`.
- **Провайдер `Attachments` недоступен** → `false` (fail-closed), а не «пропустить».
- Стойка **не читает** ModData аттачментов и не знает про `WeaponPart` напрямую — только `WeaponState.isComplete`.

### 2.3. Кэш стоек и инвалидация

Кэш локального игрока хранит `item`, `complete`, `maskConfig`, `settingsVersion`, `target`.
- При смене предмета — пересчёт.
- При смене версии настроек — пересчёт.
- **Каждый throttled-тик** пересчитывается `WeaponState.isComplete(primary)`. Если значение изменилось, кэш инвалидируется, даже если `primary` тот же самый.
- Причина: лезвие можно снять/поставить без смены предмета в руке.
- При `OnGameStart` кэши сбрасываются.

### 2.4. Запреты

- Не добавлять семейный тег без `mtmw:handle`: резолвер требует оба, но маркер роли обязателен.
- Не добавлять `mtmw:blade` «на будущее».
- Не хранить состояние лезвия в `ModData` ручки.
- Не дублировать проверку лезвия в `Resolver.lua` и `WeaponState.lua`.

## 3. Модели, текстуры и аттачменты (нативно vs MFS)

### 3.1. Нативный слой B42 (основа)

- `WeaponPart` — данные, сохранение, MP. Лезвие: `ItemType = base:weaponpart`, `PartType = Blade`, `MountOn = Base.<ручка>`.
- `ModelWeaponPart` — рендер. Формат: `<FullType лезвия> <ModelScript лезвия> <self attachment> <parent attachment>`.
  - Пример: `ModelWeaponPart = Base.MT_Attachment_Katana_Blade_Test Base.MT_Attachment_Katana_Blade_Test none Blade`.
  - Первый токен — **полный тип предмета**, не `PartType`.
  - `none` = у модели лезвия нет собственного attachment.
  - Второй токен — model script **без node transform** (scale/rotation identity). Иначе рука применяет transform ручки и transform лезвия одновременно (см. §9).
- На ручке: `WeaponSprite = <ModelScript ручки>`. Для `HandWeapon` `StaticModel` **не работает**: `HandWeapon.getStaticModel()` (`HandWeapon.java:2256`) читает `weaponSprite` инстанса, а `Item.InstanceItem` копирует только его (`Item.java:2152`). `StaticModel` остаётся для обычных предметов и `WeaponPart`.
- В model script ручки: `attachment Blade { offset = ..., rotate = ... }` — единственный источник позиции фиксированного лезвия.
- `resetEquippedHandsModels()` — обновление модели в руках после attach/detach. В MP получатель сбрасывает модель пакетом `SyncHandWeaponFields`; в SP нужен явный вызов.
- `HeadCondition` / `HeadConditionLowerChanceMultiplier` — нативная прочность (см. §4). Точную связь с `WeaponPart` проверяем в срезе.

### 3.2. MFS T (референс, не зависимость)

| Слой | Нативный B42 | MFS T |
|---|---|---|
| Хранение | `WeaponPart` на `HandWeapon` | нативный слой + отключённое ModData-зеркало |
| Установка | `MTAttachWeaponPart` / `ISRemoveWeaponUpgrade` | свой UI поверх нативных частей |
| Рендер | `ModelWeaponPart` + `attachment` | нативный рендер + `GunPos` |
| Смещение | запекается в model script ручки | per-instance offset в `GunPos` |
| Save/MP | нативные поля предмета | нативные + sync offset |
| UI | ванильное контекстное меню | 3D-инспектор |

- MFS использует нативные `getWeaponPart`, но старый ModData-зеркал `weaponpart` отключён (`EFK_AdditionalParts.lua:4`).
- `ModelScript:getAttachmentById()` возвращает **общий объект на модель**, поэтому per-instance offset надёжен только для оружия в руках. Для фиксированных лезвий это не подходит.
- MFS gun-centric: recoil, magazines, suppressors, skins. Для мили тянем только UI-паттерны и фиксы обновления модели.

### 3.3. Решение для MT_Weapons

- Вертикальный срез — только нативный T, без MFS и без `GunPos`.
- Смещение лезвия задаётся в `attachment Blade` модели ручки или в вершинах; не одновременно.
- В копии лезвия для `ModelWeaponPart` node transform (scale/rotation из FBX) обнуляется, а вершины остаются исходными. Отдельный мир-меш не нужен: `WorldStaticModel = GunCase_Ground`; уникальный мир-меш — отдельное исключение.
- Свой UI Т — позже, поверх нативных `WeaponPart`.
- Смена текстур/скинов — через `ModelAttachment` + `resetEquippedHandsModels()`; если текстура надета, базовая модель лезвия скрывается (`scale = 0.0`), оверлей (`scale = 1.0`) становится видимым.

## 4. Урон и поломка

- **Базовый урон:** нативный. `DamageModifier` лезвия прибавляется к `MinDamage`/`MaxDamage` ручки через `attachWeaponPart`.
- **Спецэффекты:** `OnWeaponHitCharacter` — только для перков/гравировок/кислоты; читаем `ModData`, не пересчитываем базовый урон.
- **Прочность:** `HeadCondition` для лезвия (нативно в B42, см. `MT_03_KNOWLEDGE` §2.7). Обычный Condition — для ручки.
- **Выпадение:** при поломке ручки аттачменты падают на землю (шанс из Sandbox).

## 5. Уникальные ручки

- Шанс выпадения (0.1%) при крафте.
- Уникальное имя + 1–2 случайные гравировки (+ и −).
- Снять гравировки нельзя.

## 6. Аттачменты и гравировки

### 6.1. Слоты (6 штук, 37 предметов)
- **Лезвие** (16 шт) — урон/прочность.
- **Текстура** (10 шт) — фикс. бонусы (Эмеральд = кислота).
- **Обмотка** (4 шт) — хват, выносливость.
- **Стержень** (7 шт) — вес, скорость, спец. урон.
- **Амулет** (3 шт) — удача, криты.
- **Подвес** (3 шт) — скорость доставания.

### 6.2. Гравировки
- UI Т (вкладка Engravings).
- Всегда плюс и минус (Волчья пасть: +5% урона / −5% скорости).
- Лимит: 3 на ручку.

### 6.3. Перки стержней
- **Серебро:** 30% шанс, что зомби не встанет.
- **Медь:** −15% выносливости при взмахе.
- **Кевлар:** не роняешь оружие при ударах.

## 7. Уход за оружием

- **Кровь:** запах привлекает зомби, ускоряет ржавчину. Смывается водой.
- **Ржавчина:** без масла теряет прочность и урон.
- **Заточка:** +15% урона на 50 ударов.
- **Оружейная стойка:** мебель для хранения, защищает от ржавчины.

## 8. Мир и лор

- **Журнал:** каталог рецептов и найденных вещей.
- **Чертежи:** записки в мире, открывают рецепты скрытых аттачей (Метеорит) и гравировок.
- **Тяжёлый замах:** зажатие ПКМ, удар по площади. *[?] под вопросом (дисмембер).*

## 9. Специфичные грабли (только MT_Weapons)

- **Стойки для мили-оружия** — через подслой `maskingright` + `m_ConditionPriority > 0`. Детали: `MT_03_KNOWLEDGE` §2.6 + §3.5.
- **Баг двуручного оружия в одной руке:** движок стирает маску на 1–2 кадра. Лечится Watchdog на 5 тиков.
- **Анимация 10–20 кадров:** `<m_SpeedScale>1.00</m_SpeedScale>`, `<m_Looped>true</m_Looped>`. Захватывать только кости рук (и спины при необходимости), ноги — ванили.
- **Скелет пальцев PZ:** 2 кости на руку (`Finger0`, `Finger1`), симметрично.
- **Ходьба не считается `busy`:** `player:isCurrentActionPathfinding()` (см. `MT_03_KNOWLEDGE` §2.5).
- **MP-синхронизация стоек:** передаём `MT_MeleeMaskIdle`, `MT_MeleeMaskAim` и флаг `MT_MeleeBusy`. Никакого `gsub("Idle", "Aim")`: aim-маска приходит готовой. Получатель выбирает `p:isAiming() and aim or idle`. Локальный игрок — единственный писатель своих ModData; удалённые только читают.
- **`ModelWeaponPart`:** первый токен — полный тип предмета лезвия (`Base.<Blade>`), а не `PartType`.
- **`PartType` — свободная строка:** `Blade` работает без отдельного enum. `getWeaponPart("Blade")` ищет по этой строке.
- **`attachWeaponPart`:** добавляет `DamageModifier` лезвия к `MinDamage`/`MaxDamage` ручки. Это нативно, свой расчёт урона для базового слоя не нужен.
- **Двойной transform лезвия в руке:** рука применяет mesh transform ручки, а затем mesh transform дочернего лезвия. В тестовых FBX у обоих `Lcl Scaling = 100` и `Lcl Rotation = 90`, поэтому лезвие получает 100×100 и улетает. Инвентарный путь (`ItemModelRenderer.initWeaponPart`) transform ребёнка не применяет, но transform ручки применяет. Поэтому для `ModelWeaponPart` нужна копия с **identity node transform и исходными вершинами**: тогда transform ручки применяется ровно один раз в обоих путях. Запекать transform в вершины нельзя — это снова даёт 100×100. Отдельный мир-меш не нужен: `WorldStaticModel = GunCase_Ground`; уникальный мир-меш — отдельное исключение.
- **Контекстное меню ищет части только в основном инвентаре:** `ISInventoryPaneContextMenu.lua:763` использует `getItemsFromCategory`, без рекурсии. `AttachmentMenu.lua` добавляет `getAllCategoryRecurse` и переиспользует ванильную опцию, если она уже есть.
- **Ванильный attach перемещает оружие в инвентарь:** `onUpgradeWeapon` сначала экипирует лезвие и отвёртку, из-за чего оружие уходит из рук. Если начать идти или отменить действие до `ISUpgradeWeapon`, остаётся полусостояние: оружие в инвентаре, лезвие экипировано/потеряно, аттачмента нет. `MTAttachWeaponPart.lua` заменяет колбэк для всех MT-ручек: переносит только часть в основной инвентарь, не трогает руки, а в `complete()` атомарно вызывает `attachWeaponPart`, синхронизацию и удаляет часть. Отмена до `complete()` ничего не меняет. Если перенос части из сумки прерван, `isValid()` не даёт действию прикрепить лезвие из сумки: отмена не оставляет аттачмента. В MP `complete()` выполняет сервер через `NetTimedAction`, поэтому файл лежит в `shared`, а `syncHandWeaponFields`/`sendRemoveItemFromContainer` работают штатно.
- **Отладочные логи:** `DamageDebug.lua` слушает `OnWeaponHitCharacter` и печатает `damageSplit`, урон оружия, лезвие и HP до/после. Все сообщения MT идут через `MT_Debug` с подсистемными тегами `[Module - Weapon]`, `[Module - Damage]`, `[Module - Stance]`; включается опцией `DebugEnable` на странице `MT_MeleeWeapon`.
- **Обновление модели в SP:** `ISUpgradeWeapon`/`ISRemoveWeaponUpgrade`/`MTAttachWeaponPart` меняют аттачмент в `complete()`, а не в `perform()`, и не гарантируют `resetEquippedHandsModels()`. У ванильного `ISUpgradeWeapon` в `complete` есть `setSecondaryHandItem(nil)`, но он сбрасывает модель только при реальной смене; у `MTAttachWeaponPart` этого вызова нет — руки не трогаются. `AttachmentRefresh.lua` оборачивает `complete` у всех трёх классов и вызывает `resetEquippedHandsModels()` сразу после изменения.
- **MP-обновление модели:** `SyncHandWeaponFieldsPacket` на получателе сам вызывает `resetEquippedHandsModels()`.

## 10. Файлы мода

### 10.1. Lua-файлы

`MT_Stance` — данные отдельно от логики. PZ сортирует полный путь: и `MT_MeleeStances_Config.lua`, и `MT_MeleeStances_Data.lua` идут раньше подпапки `Runtime/` (`M` < `R`). Подпапка `Logic/` шла бы раньше `MT_MeleeStances_Data.lua` (`L` < `M`), то есть логика загрузилась бы до данных. Страница настроек создаётся в `MT_ModOptions.lua` отложенно (`OnGameBoot`/`OnGameStart`), а `MT_Stance/Runtime/Options.lua` только регистрирует в ней свои опции; поэтому Config и Data обязаны грузиться раньше Runtime.

```
media/lua/client/MT_ModOptions.lua   — единственный владелец страницы PZAPI, очередь регистрации

media/lua/client/MT_Debug/
└── MT_Debug.lua                      — клиентский логгер, подсистемные теги

media/lua/client/MT_Stance/
├── MT_MeleeStances_Config.lua        — все константы (пороги, ключи ModData, позы)
├── MT_MeleeStances_Data.lua          — Lists, Categories, Subs (только данные)
└── Runtime/
    ├── Lifecycle.lua                 — подписки событий, сброс кэшей
    ├── LocalController.lua           — локальный игрок, watchdog, re-assert, stance-target
    ├── Options.lua                   — регистрация стоек через MT_Options
    ├── RemoteController.lua          — удалённые игроки, чтение масок
    ├── Resolver.lua                  — тег → категория/подкатегория → маска
    ├── Settings.lua                  — чтение и кэш настроек по версии
    └── Sync.lua                      — запись idle/aim/busy в ModData + transmit
```

`Lifecycle.lua` загружается первым внутри `Runtime` (`L`). Он не захватывает соседние модули на верхнем уровне — только разрешает их в момент события.

`MT_ModOptions.lua` (`M`) грузится после `MT_Debug` (`D`) и до `MT_Stance` (`S`). Все модули регистрируют опции через `MT_Options.register`; страница создаётся отложенно (`OnGameBoot`/`OnGameStart`), поэтому порядок не ломает загрузку. `MT_Debug` добавляет секцию последней — внизу страницы, после `MT_Options.separator()`.

Общий контракт оружия — в `shared`, чтобы грузиться раньше клиентской логики:

```
media/lua/shared/MT_MeleeWeapons_Common/
├── Tags.lua                          — доступ к ItemTag, hasTag с логом
├── Attachments.lua                   — нативный getWeaponPart("Blade")
├── MTAttachWeaponPart.lua            — timed action; в shared, иначе MP-сервер не найдёт класс
└── WeaponState.lua                   — isComplete(item)
```

`MTAttachWeaponPart.lua` обязан лежать в `shared`: в MP клиент отправляет `NetTimedAction` с именем класса, и сервер восстанавливает действие через `MTAttachWeaponPart.new(character, weapon, part)`. В `client` сервер класс не загрузит.

`media/registries.lua` остаётся ранней регистрацией `ItemTag` (`MTMWRegistries.ItemTags`); `Tags.lua` ничего не регистрирует, только читает.

Клиентская логика среза:

```
media/lua/client/MT_Weapons/Runtime/
├── AttachmentMenu.lua                — части из сумок/вложенных контейнеров
├── AttachmentRefresh.lua             — resetEquippedHandsModels после attach/detach + логи
└── DamageDebug.lua                   — OnWeaponHitCharacter, damageSplit и HP до/после
```

### 10.2. Скрипты предметов

```
media/scripts/
├── Axe/
├── Katana/
│   ├── MT_Katana_Test.txt                 — item ручки
│   └── MT_Katana_Test_Attachments.txt     — model ручки + attachment points
├── Knife/
├── OneHandAxe/
├── Attachment/
│   ├── Blade/
│   │   └── Katana/
│   │       ├── MT_Attachment_Katana_Blade_Test.txt
│   │       └── MT_Attachment_Katana_Blade_Test_Attachments.txt
│   ├── Texture/
│   ├── Wrap/
│   ├── Core/
│   ├── Amulet/
│   └── Suspension/
└── Holsters/
    ├── Axes/MT_Holster_Axes.txt
    ├── Katanas/MT_Holster_Katana_*.txt
    ├── Knifes/MT_Holster_Knifes.txt
    ├── OneHandAxes/MT_Holster_OneHandAxes.txt
    ├── MT_Holster_Attachment.txt
    └── MT_Holster_Recipie.txt
```

Правило: один слот = один `<PartType>`. Все скрипты аттачментов лежат в `scripts/Attachment/<PartType>/<Category>/`, а не в папке категории ручки. Папки `Texture`, `Wrap`, `Core`, `Amulet`, `Suspension` — задел на остальные слоты.

### 10.3. Анимации (animsets)

```
media/animsets/player/maskingright/
├── MT_AxeAim.xml, MT_AxeIdle.xml
├── MT_KatanaAim.xml, MT_KatanaAim_Tactical.xml
├── MT_KatanaIdle.xml, MT_KatanaIdle_Tactical.xml
├── MT_KnifeAim.xml, MT_KnifeIdle.xml
└── MT_OneHandAxeAim.xml, MT_OneHandAxeIdle.xml
```

**Правило:** `m_AnimName` в animset обязан ссылаться на существующий клип в `media/anims_x/Bob/`. Если клипа нет, движок молча играет дефолтную анимацию, и это выглядит как «маска слетела».

- `MT_KatanaIdle` → `Bob_MT_KatanaIdle` (есть).
- `MT_KatanaAim` → `Bob_TAC` (временно; отдельный `Bob_MT_KatanaAim.fbx` — TBD).
- Остальные aim-ноды используют `Bob_TAC` / `Bob_TACHighReady`.

Aim-маска не вычисляется через `gsub("Idle", "Aim")`: `MT_MeleeMaskIdle` и `MT_MeleeMaskAim` передаются готовыми, получатель выбирает по `isAiming()`.

### 10.4. Модели и текстуры

- Ручки: `media/models_X/<Category>/<Handle>.fbx`, `media/textures/<Category>/<Handle>.png`.
- Аттачменты: `media/models_X/Attachment/<PartType>/<Category>/<Part>.fbx`, `media/textures/Attachment/<PartType>/<Category>/<Part>.png`.
- `mesh` в model script — путь относительно `models_X`, без расширения.
- `texture` в model script — путь относительно `textures`, без расширения.
- Иконки — `media/textures/item_<Icon>.png` (128×128).
- Суффикс `_Attached` **не используется**. Если у лезвия несколько вариантов под ручки, имя = `<PartItem>_<HandleSuffix>`; при одном варианте модель может совпадать с `<PartItem>`.

Тестовый срез:

| Файл | Назначение |
|---|---|
| `scripts/Katana/MT_Katana_Test.txt` | item ручки |
| `scripts/Katana/MT_Katana_Test_Attachments.txt` | model ручки + точки `world`/`Blade`/`Bip01_Prop2` |
| `scripts/Attachment/Blade/Katana/MT_Attachment_Katana_Blade_Test.txt` | item лезвия |
| `scripts/Attachment/Blade/Katana/MT_Attachment_Katana_Blade_Test_Attachments.txt` | model лезвия для `ModelWeaponPart` |
| `models_X/Katana/MT_Katana_Test.fbx` | меш ручки |
| `models_X/Attachment/Blade/Katana/MT_Attachment_Katana_Blade_Test.fbx` | меш лезвия с identity node transform |
| `textures/Katana/MT_Katana_Test.png` | текстура ручки, 512×512 |
| `textures/Attachment/Blade/Katana/MT_Attachment_Katana_Blade_Test.png` | текстура лезвия, 512×512 |
| `textures/item_MT_Katana_Test.png` | иконка ручки, 128×128 |
| `textures/item_MT_Attachment_Katana_Blade_Test.png` | иконка лезвия, 128×128 |

### 10.5. Контракт тестовых скриптов (эталон шаблона)

Полные шаблоны: `templates/HANDLE_TEMPLATE.txt` и `templates/ATTACHMENT_TEMPLATE.txt`.

Ручка:
```ini
item MT_Katana_Test
{
    ItemType = base:weapon,
    Tags = mtmw:handle;mtmw:katana,
    WeaponSprite = MT_Katana_Test,
    WorldStaticModel = MT_Katana_Test,
    ModelWeaponPart = Base.MT_Attachment_Katana_Blade_Test Base.MT_Attachment_Katana_Blade_Test none Blade,
}
```

`WeaponSprite` обязателен для `HandWeapon`: без него `getStaticModel()` возвращает null, и `ModelManager` падает в `ScriptManager.getItemName`.

Лезвие:
```ini
item MT_Attachment_Katana_Blade_Test
{
    ItemType = base:weaponpart,
    PartType = Blade,
    MountOn = Base.MT_Katana_Test,
    DamageModifier = 2.0,
    WeightModifier = 1.0,
    WorldStaticModel = GunCase_Ground,
}
```

Модель ручки:
```ini
model MT_Katana_Test
{
    mesh = Katana/MT_Katana_Test,
    texture = Katana/MT_Katana_Test,
    attachment world { offset = 0.0 0.0 0.0, rotate = 0.0 0.0 0.0, }
    attachment Blade { offset = 0.0 0.0 0.0, rotate = 0.0 0.0 0.0, }
    attachment Bip01_Prop2 { offset = 0.0 0.0 0.0, rotate = 0.0 0.0 0.0, }
}
```

Модель лезвия — без attachment, только `mesh` + `texture`:
```ini
model MT_Attachment_Katana_Blade_Test
{
    mesh = Attachment/Blade/Katana/MT_Attachment_Katana_Blade_Test,
    texture = Attachment/Blade/Katana/MT_Attachment_Katana_Blade_Test,
}
```

Правила:

- `attachment world` — корневая точка для мира/превью.
- `attachment Blade` — точка стыка; `ModelWeaponPart` ссылается на неё как `none Blade`.
- `attachment Bip01_Prop2` — хват левой рукой; ставится на рукоять, не на клинок.
- `MountOn` — **одна** строка с `;`-списком всех ручек категории. Повторный `MountOn` перезаписывает список.
- `ModelWeaponPart` — одна строка на каждый совместимый item лезвия. Токены: `<FullType лезвия> <model script для этой ручки> none Blade`.
- Если лезвие есть в `MountOn`, но для ручки нет `ModelWeaponPart`, лезвие прикрепится логически и не отрисуется.
- `_Attached` в имени больше не используется: `WorldStaticModel = GunCase_Ground`, а для `ModelWeaponPart` — модель `<PartItem>_<HandleSuffix>` (или `<PartItem>`, если вариант один).
- Ванильные теги ручек и аттачментов — TBD; в шаблонах их нет.
- `base:weaponpart` — не `HandWeapon`: снятое лезвие, взятое в руку как отдельный предмет, не имеет hand-модели. В инвентаре у него только иконка, в мире — `WorldStaticModel` (`GunCase_Ground`). `ModelWeaponPart` рендерит часть только в составе ручки; это ожидаемо. Если когда-нибудь понадобится отдельная модель в руке, это будет отдельный `WeaponSprite`/hand-model, а не система аттачментов.

### 10.5.1. Экспорт FBX для ручек и вставных частей

**Почему лезвие может стать 100×.** В руке движок применяет transform ручки, а затем transform узла каждой `ModelWeaponPart`. Если у лезвия в FBX тоже стоит `Lcl Rotation = 90` и `Lcl Scaling = 100` (как у ручки), transform применяется дважды: лезвие получает 100×100 и улетает. В инвентарном превью путь другой, поэтому один и тот же FBX может выглядеть нормально в мире и неправильно в руке.

**Правило для `ModelWeaponPart`:**

- узел `Model` в FBX должен быть identity: `Lcl Translation = 0,0,0`, `Lcl Rotation = 0,0,0`, `Lcl Scaling = 1,1,1`;
- вершины — в локальной системе ручки, без запекания transform ручки;
- `attachment Blade` в model script — отдельная точка стыка; её offset/rotate не заменяют transform узла.

Для самой ручки текущий transform (`90/100` у тестовой) допустим: движок применяет его один раз. Проблема только у дочерней части, которая получает transform ручки **и** свой.

**Экспорт в Blender:**

1. Делать лезвие в той же сцене и в том же масштабе, что ручку, в её локальных координатах.
2. Для каждой ручки, на которую ставится это лезвие, сделать отдельную копию объекта. Имя — `<PartItem>_<HandleSuffix>`, без `_Attached`; если вариант один, можно `<PartItem>`.
3. У копии обнулить transform объекта: Location 0, Rotation 0, Scale 1. **Не применять** старый transform к вершинам (`Apply All Transforms` / `Apply Rotation & Scale` не использовать): иначе 90/100 запечётся в вершины, и движок снова применит transform ручки сверху. В Blender копия может визуально сместиться — это ожидаемо: в игре её transform возьмёт ручка.
4. Экспортировать **только выбранный объект** (Selected Objects), без ручки, камер и света, в FBX Binary с теми же осями и unit scale, что у существующих ручек (для тестовых ассетов — Y-up, unit scale 1.0). Главное — не включить опцию, которая запекает object transform в вершины; после экспорта проверить узел.
5. Отдельный мир-меш не нужен: `WorldStaticModel = GunCase_Ground`. Если позже понадобится уникальный вид на земле — добавить отдельную модель и явно указать её в `WorldStaticModel`, это исключение.
6. В `_Attachments` аттачмента добавить модель без attachment и указать её в ручке:
   ```ini
   model MT_Attachment_Katana_Blade_T1_Yamato
   {
       mesh = Attachment/Blade/Katana/MT_Attachment_Katana_Blade_T1_Yamato,
       texture = Attachment/Blade/Katana/MT_Attachment_Katana_Blade_T1_Yamato,
   }
   ```
   ```ini
   ModelWeaponPart = Base.MT_Attachment_Katana_Blade_T1 Base.MT_Attachment_Katana_Blade_T1_Yamato none Blade
   ```
7. Проверить FBX: у узла `Model` `Lcl Rotation = 0,0,0`, `Lcl Scaling = 1,1,1`. `Lcl Translation` — 0, если смещение задаётся через `attachment` или вершины; иначе лезвие уедет.
8. Проверка в игре: в руке и в 3D-превью лезвие одного размера с ручкой, не 100×, на своём месте. Если 100× — у узла остался 90/100; если смещено — смотреть `attachment Blade` offset/rotate, а не масштаб.

**Частые ошибки:**

- суффикс `_Attached` в имени: он больше не нужен; имя = `<PartItem>_<HandleSuffix>`;
- один FBX на несколько ручек: у каждой ручки свой меш/текстура/положение, значит своя model-запись;
- запекание 90/100 в вершины: не помогает, transform ручки всё равно применяется сверху;
- `Lcl Translation` из Blender: лезвие уезжает; смещение лучше задавать в `attachment Blade` или в вершинах;
- забытый `WeaponSprite` у ручки: `getStaticModel()` вернёт null.

### 10.5.2. Структура контента и шаблоны

Канон папок:

```
media/scripts/<Category>/<Handle>.txt                    — item ручки
media/scripts/<Category>/<Handle>_Attachments.txt        — model ручки + точки
media/scripts/Attachment/<PartType>/<Category>/<Part>.txt — item аттачмента
media/scripts/Attachment/<PartType>/<Category>/<Part>_Attachments.txt — model аттачмента
media/models_X/<Category>/<Handle>.fbx
media/models_X/Attachment/<PartType>/<Category>/<Part>.fbx
media/textures/<Category>/<Handle>.png
media/textures/Attachment/<PartType>/<Category>/<Part>.png
media/textures/item_<Icon>.png
```

Правила деления:

- Ручка и её модель — два файла: item и `_Attachments` (model + точки).
- Аттачмент и его модели — два файла: item и `_Attachments`.
- Один item аттачмента на категорию/тир. Визуалы для разных ручек — отдельные модели `model <PartItem>_<HandleSuffix>` в `_Attachments`, без `_Attached`. Если вариант один, модель может называться `<PartItem>`.
- `ModelWeaponPart` живёт в скрипте ручки, а не в скрипте аттачмента.
- `WorldStaticModel = GunCase_Ground` — только для аттачментов. Для ручки — её собственный FBX.
- `WorldStaticModel` — не hand-модель. Снятый `weaponpart` в руке не отображается; это норма, а не ошибка `ModelWeaponPart`.
- Имена: ручки `MT_<Category>_<Name>`, аттачменты `MT_Attachment_<Category>_<PartType>_<Name>`.
- `Attachment` — единственное число; `Balde`/`Attachemnt` — опечатки, не использовать.

Шаблоны для копирования лежат в `MT_DOCS/MT_MODS/MT_Weapons/templates/`, не в `media/` (иначе PZ попытается загрузить их как скрипты).

### 10.6. Чек-лист вертикального среза

Сейчас (тестовая ручка `MT_Katana_Test` с `mtmw:katana`; стойка включается только при наличии лезвия `MT_Attachment_Katana_Blade_Test`):
1. Прикрепить лезвие из основного инвентаря: модель появилась в руке и в 3D-превью инвентаря.
2. Прикрепить лезвие из сумки или вложенной сумки: опция появилась, лезвие перенеслось и прикрепилось.
3. `MinDamage`/`MaxDamage` выросли на `DamageModifier` лезвия; в логе `[Module - Weapon] attach` видно `0.000..0.000 -> 2.000..2.000`.
4. Удар по зомби: в логе `[Module - Damage] hit` есть `split` и лезвие, в `hit-result` — HP до/после; HP падает.
5. Прикреплённое лезвие в руке и в 3D-превью нормального размера и на своём месте (не 100×). Снятое лезвие как отдельный `weaponpart` в руке не рендерится — это ожидаемо.
6. Снять лезвие: модель исчезла, урон вернулся к 0; в логе `[Module - Weapon] detach` видно `2.000..2.000 -> 0.000..0.000`.
7. Save/load: лезвие осталось прикреплённым.
8. MP: клиент прикрепил лезвие — второй клиент видит модель; после реджойна состояние сохранилось.
9. Отмена или начало ходьбы во время установки лезвия: оружие остаётся там, где было (в руках или в инвентаре), лезвие не теряется, аттачмента нет, повторная установка работает.
10. Установка при оружии в руках и при оружии в инвентаре: оружие не перекладывается, после установки остаётся в том же месте.

После рефактора `MT_Stance` и добавления `mtmw:katana` (гейт реализован):
11. Ручка без лезвия стойку не получает.
12. Снятие лезвия снимает стойку в течение throttle-окна.
13. Смена предмета в руке не оставляет чужую маску.
14. `Vanilla` в настройках снимает маску.
15. Прицеливание: в логе `stance-target` есть `target=MT_KatanaAim reason=aim`, маска не падает в дефолт; после отпускания — `target=MT_KatanaIdle reason=idle`.
16. Атака во время прицеливания: маска снимается на удар и возвращается в `MT_KatanaAim` после него.

**Статус SP (25.10.2026):** пункты 1–7 и 9–10 подтверждены. Пункт 8 (MP) не проверен. Пункты 11–16 готовы к проверке: рефактор `MT_Stance` выполнен, `MT_KatanaAim` переведён на `Bob_TAC`, тестовые скрипты мигрированы под шаблон.

### 10.6.1. MP: статус и что проверять

MP — **целевая платформа** мода, а не опция. В текущем срезе MP-путь реализован, но **не проверен**: нет тестового сервера и второго клиента. Пункт 8 нельзя считать выполненным до отдельного теста.

Что уже заложено:

- `MTAttachWeaponPart.lua` лежит в `shared`, чтобы сервер восстановил `NetTimedAction` по имени класса;
- `complete()` на сервере вызывает `attachWeaponPart`, `syncHandWeaponFields` и `sendRemoveItemFromContainer`;
- `AttachmentRefresh.lua` — клиентский; в MP модель обновляется пакетом `SyncHandWeaponFields`;
- нативный `WeaponPart` хранится в предмете, поэтому после реджойна состояние должно приходить с сервера;
- стойки передают `MT_MeleeMaskIdle`, `MT_MeleeMaskAim` и `MT_MeleeBusy` через ModData; этот путь тоже не проверен в MP.

Что проверить, когда появится стенд:

1. Клиент A прикрепляет лезвие из основного инвентаря и из сумки; клиент B видит модель и урон.
2. Клиент A снимает лезвие; клиент B видит снятие.
3. Реджойн клиента A: лезвие остаётся прикреплённым, модель в руках и в превью.
4. Отмена или ходьба во время установки на клиенте A: сервер не получает аттачмент, лезвие не пропадает.
5. Вложенная сумка: после установки лезвие не остаётся «призраком» в сумке у клиента B.
6. Логи через `MT_Debug` на клиенте и сервере: `[Module - Damage] hit` с `mode=MP-client` / `MP-server`, `[Module - Weapon] attach-start`, `[Module - Weapon] attach`, `[Module - Weapon] detach`, `[Module - Stance] stance-target`.

До появления стенда MP-поведение считается **непроверенным**; фидбек с серверов будем собирать после релиза.

### 10.7. Отладка

Страница настроек: заголовок `[MT] Melee Weapon` (`UI_MT_MeleeWeapon_Title`). Под ним — сепаратор, затем секция `Стойки холодного оружия` (`UI_MT_MeleeStances_Title`), два вертикальных отступа, затем `GlobalEnable` и категории. Дебаг — последним, после своего сепаратора.

Клиентский логгер — `media/lua/client/MT_Debug/MT_Debug.lua`. Все сообщения идут через `MT_Debug.log(tag, fmt, ...)` и включаются опцией `DebugEnable` на странице `MT_MeleeWeapon` (PZAPI ModOptions, по умолчанию выключено). Страница принадлежит `media/lua/client/MT_ModOptions.lua`; `MT_Stance/Runtime/Options.lua` только регистрирует стойки, а `MT_Debug.registerOptions` добавляет `DebugEnable` последней — внизу страницы, после `MT_Options.separator()`. `MT_Debug.isEnabled()` читает `MT_Options.get("DebugEnable")`; при отсутствии страницы/опции логи выключены.

Теги: `[Module - Weapon]` — аттачменты и меню, `[Module - Damage]` — удары, `[Module - Stance]` — стойки. Движок сам добавляет префикс мода, поэтому фактический вид — `[[MT] Melee Weapons] [Module - Weapon] ...`. Формат `[[MT] Melee Weapons [Module - Weapon]]` из Lua не получается: закрывающая скобка принадлежит движку; `Console:Echo` в игре отсутствует.

## 11. Открытые вопросы
- [?] Список 18 ручек (какие модели).
- [?] Тяжёлый замах + дисмембер.
- [?] Токен смены имени.
- [?] Метание мили-оружия.
- [?] Зазубрины при 0 прочности.
- [?] Точная связь `HeadCondition` и `WeaponPart` (что считать прочностью лезвия).
- [?] Нужен ли `mtmw:blade` позже, когда появятся лезвия без `WeaponPart`.
- [?] Формат 10 текстур лезвий: отдельные `WeaponPart`, `ModelAttachment` или скин.
- [?] Минимальный UI Т для мили: какие вкладки нужны кроме Attachments.
- [?] Отдельный клип `Bob_MT_KatanaAim.fbx` вместо временного `Bob_TAC`.