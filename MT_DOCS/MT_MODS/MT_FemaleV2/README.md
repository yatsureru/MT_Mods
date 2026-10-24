# MT_MODS/MT_FemaleV2/README.md

# [MT] Female v2 — README

> Фичи и TODO. Технические детали — `MT_MODS/MT_FemaleV2/DATABASE.md`.

**Steam Workshop ID:** `3472993929`

## 1. Возможности

**Next Gen Female Body** — замена 3D-модели женского тела. Две версии:

1. **Standard (FBX Replacer)** — простая замена файлов на диске. Для мощных ПК и SP без толп зомби. Не требует ZombieBuddy.
2. **Optimized (Java Patch)** — динамическая подмена через `ModelInstance.init`. Игрок и NPC получают high-poly модель, зомби остаются на ванильной. Снижает нагрузку на GPU и шину памяти. Требует ZombieBuddy.

## 2. TODO

- [x] Java-патчи загрузки модели в память (`Patch_LoadCustomModel`).
- [x] Патч выборочной подмены для персонажей (`Patch_PlayerModel`).
- [x] Патчи для UI превью (`Patch_PreviewModel`, `Patch_PreviewSurvivor`).
- [x] Патч подмены текстур в `PopTemplateManager` (`Patch_SkinTexture`).
- [ ] Обновлять патчи при выходе новых билдов B42 (следить за `ModelManager`, `ModelInstance`).

## 3. Идеи / Backlog
- [ ] Опция в ModOptions: «Отключить кастомную модель в MP».
- [?] Синхронизация кастомной модели для NPC в MP.