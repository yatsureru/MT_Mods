# MT_MODS/MT_FemaleV2/DATABASE.md

# [MT] Female v2 — DATABASE

> **[MT] AI reminder:** не гадай API — запрашивай декомпил (`MT_04_NAVIGATION` §1). Ищи идиому до примитива. Помечай `[?]` предположения. Полный протокол — `MT_01_AI_PROTOCOL`.
> **Общие правила:** `MT_02_RULES`. **Паттерны Java-патчей:** `MT_03_KNOWLEDGE` §3.4.

## 1. Архитектура

**Стек:** Java-патчи (ZombieBuddy) + `.fbx` + `.png`.

### 1.1. Standard (FBX Replacer)
- Прямая замена файлов на диске.
- Не требует ZombieBuddy.
- Минус: all-in (игрок + NPC + зомби).

### 1.2. Optimized (Java Patch)
- **Загрузка:** `ModelManager.create` — модель загружается в ОЗУ при старте.
- **Подмена:** перехват `ModelInstance.init`, подмена указателя `mi.model` только для `!chr.isZombie()`.
- **Текстуры:** подмена через `PopTemplateManager.femaleSkins`.
- **UI превью:** отдельные патчи для `Patch_PreviewModel`, `Patch_PreviewSurvivor`.
- Требует ZombieBuddy.

### 1.3. Патчи (по `MT_MODS_TODO` §9.2)
- `Patch_LoadCustomModel` — загрузка модели в память.
- `Patch_PlayerModel` — выборочная подмена для персонажей (`!isZombie`).
- `Patch_PreviewModel`, `Patch_PreviewSurvivor` — UI превью.
- `Patch_SkinTexture` — текстуры в `PopTemplateManager`.

## 2. Специфичные грабли
- **Зомби на ванильной модели** — принципиально для Optimized (экономия).
- **Обновление при патчах игры:** `ModelManager` и `ModelInstance` могут измениться между билдами — патчи требуют проверки.
- **MP:** `[?]` синхронизация кастомной модели для NPC не проработана.

## 3. Открытые вопросы
- [?] Есть ли дизайн-док с деталями (какие модели, как комбинируются, что с анимациями)?
- [?] Как ведёт себя модель при смерти зомби (переход на ванильный ragdoll)?
- [?] Совместимость с другими модами на модели?