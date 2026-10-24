# [MT] Project Repository

> Точка входа в репозиторий проекта `[MT]` (Project Zomboid Build 42.21).
> **Для ИИ-агентов:** документация проекта начинается с `MT_DOCS/MT_00_INDEX.md`. Не читай декомпил целиком — навигация по задачам описана там.

## 1. Что здесь лежит

Репозиторий хранит три независимых слоя:

```
Repo/
├── Java_Decompile/                    ← декомпил Java-слоя игры (zombie.*)
│   ├── zombie/                        ← сами .java файлы, структура = пакеты
│   ├── Java_Decompile_commands_for_tree/  ← команды генерации дерева
│   └── Java_Decompile_tree.md         ← дерево Java-слоя (навигация)
│
├── Lua_Decompile/                     ← декомпил Lua-слоя игры (media/lua/*)
│   ├── lua/                           ← сами .lua файлы (client/shared/server)
│   ├── Lua_Decompile_commands_for_tree/   ← команды генерации дерева
│   └── Lua_Decompile_tree.md          ← дерево Lua-слоя (навигация)
│
├── MT_DOCS/                           ← документация проекта [MT]
│   ├── MT_00_INDEX.md                 ← точка входа для ИИ и людей
│   ├── MT_01_AI_PROTOCOL.md           ← как работает ИИ (процесс)
│   ├── MT_02_RULES.md                 ← §1 факты движка / §2 соглашения
│   ├── MT_03_KNOWLEDGE.md             ← §1 симптомы / §2 факты API / §3 паттерны
│   ├── MT_04_NAVIGATION.md            ← навигация по декомпилу + чек-листы
│   └── MT_MODS/                       ← README + DATABASE по каждому моду
│
└── README.md                          ← этот файл
```

## 2. Три слоя — три роли

| Слой | Что | Как использовать |
|---|---|---|
| **Java_Decompile/** | Исходники Java-слоя игры (`zombie.*`). Декомпил Vineflower v0.3.2. | Когда нужен факт по Java-API (например, `ItemContainer.AddItem`). Сначала `Java_Decompile_tree.md` — найти файл, потом открыть конкретный `.java`. |
| **Lua_Decompile/** | Исходники Lua-слоя игры (`media/lua/`). Ванильные `IS*Action`, `IS*Panel`, `ISBaseTimedAction` и т.д. | Когда нужна идиома Lua-слоя (например, `ISWearClothing`, `ISTimedActionQueue`). Сначала `Lua_Decompile_tree.md`, потом конкретный `.lua`. |
| **MT_DOCS/** | Документация **нашего** проекта `[MT]`. Правила, факты, паттерны, по-модовые доки. | Всегда. Начинать с `MT_00_INDEX.md` → карта «задача → файлы». |

**Важно:** `Java_Decompile/` и `Lua_Decompile/` — это **ваниль игры**, не наши моды. Наши моды — отдельные папки в воркспейсе (не в этом репозитории).

## 3. Для ИИ-агентов

**Порядок загрузки:**
1. `MT_DOCS/MT_00_INDEX.md` — карта задач и модов.
2. `MT_DOCS/MT_01_AI_PROTOCOL.md` — 5 правил (грузится всегда).
3. Дальше — **по карте** из `MT_00_INDEX` §2. Не тяни всё подряд.

**Правила запроса факта:**
- Java-API → `Java_Decompile/zombie/<полный/путь/Файл>.java:<строка>`.
- Lua-API → `Lua_Decompile/lua/<client|shared|server>/.../<Файл>.lua:<строка>`.
- Всегда **полный путь**, не только имя класса — одноимённые классы бывают в разных пакетах.

**Версия декомпила:** Build 42.21 (Vineflower v0.3.2, Zomboid Decompiler).

## 4. Обновление декомпила

При выходе нового патча игры:

1. Перегенерировать `Java_Decompile/zombie/` — команды в `Java_Decompile/Java_Decompile_commands_for_tree/`.
2. Перегенерировать `Lua_Decompile/lua/` — команды в `Lua_Decompile/Lua_Decompile_commands_for_tree/`.
3. Перегенерировать `Java_Decompile_tree.md` и `Lua_Decompile_tree.md`.
4. **Сверить с `MT_DOCS/MT_02_RULES` / `MT_03_KNOWLEDGE`** — если факты изменились, обновить канон.
5. Обновить Changelog в `MT_02` / `MT_03` / `MT_04`.

## 5. Соглашения

- **Версия игры:** 42.21. Не путать с 41 — API отличается.
- **Декомпил не редактируется.** Правки только в `MT_DOCS/`.
- **Деревья (`*_tree.md`) — навигация, не источник истины.** Источник — сами `.java` / `.lua`.