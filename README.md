# Шаблон репозитория аддона GlobalWar

> **English summary.** This repo is the catalog entry of one GlobalWar addon: `addon.ltx` (the manifest) and
> `catalog/` (the card shown in the launcher). The addon files themselves are **not** in git: every version is a
> GitHub Release with the package `<id>-<version>.zip` attached as an asset. Fill `addon.ltx` and `catalog/`,
> check them with `gw_addon_tool.py validate-repo`, pack the addon with `gw_addon_tool.py pack`, publish a release tagged
> `v<version>`. The central CI of the index validates it again and publishes it (or opens an Issue with the errors).

## Что это за репозиторий

Один репозиторий = один аддон каталога. Имя репозитория равно `id` аддона. Здесь лежат только:

- `addon.ltx` — манифест аддона (тот же файл, что в пакете последней версии);
- `catalog/` — карточка аддона в лаунчере: название, описание, иконка, скриншоты.

**Файлы аддона (`gamedata/` и т.д.) в git не кладутся.** Каждая версия — это GitHub Release, к которому
прикреплён zip-пакет. Так ограничения GitHub на размер репозитория не мешают большим пакам: лимит есть только на один
файл релиза (меньше 2 ГиБ), и большой пакет режется на части.

Исходники разработки (рабочая папка аддона, исходные текстуры, проекты SDK, Blender) могут жить где угодно: в другом
репозитории, на диске, в облаке. Лаунчер их не читает.

## Раскладка

| Путь | Обязательно | Что это |
|---|---|---|
| `addon.ltx` | да | манифест; `id` = имя репо |
| `catalog/catalog.ltx` | да | поля карточки: название, краткое описание, теги, авторы, лицензия, ссылки |
| `catalog/description.md`, `catalog/description.<язык>.md` | хотя бы один | описание аддона (markdown) |
| `catalog/icon.png` | нет | иконка 256x256 |
| `catalog/screenshots/` | нет | до 10 скриншотов |
| `.github/workflows/validate.yml` | нет | проверка `addon.ltx` и `catalog/` при push |
| `.github/workflows/release.yml` | нет | проверка ассетов при публикации релиза |
| `README.md`, `LICENSE` | нет | для GitHub; лаунчер их не читает |

## 1. Заполнить `addon.ltx`

Если репо создал бот каталога, плейсхолдеры `{{...}}` уже заполнены. Если вы создали репо сами — заполните их:
оставшийся `{{...}}` — ошибка `repo_placeholder`. Все поля манифеста описаны в вики основного проекта:
`wiki/doc/plugins/manifest.md`.

```ini
; Комментарий в addon.ltx - от ";" или "//" до конца строки. Движок режет КАЖДОЕ "//",
; поэтому ссылки (https://...) в addon.ltx не пишите - для них есть catalog/catalog.ltx.

[addon]
id       = my_weapons   ; = имя репо: только a-z 0-9 _, до 64 символов. Иначе manifest_id_mismatch
version  = 1.2.0        ; три числа MAJOR.MINOR.PATCH, без суффиксов (-beta нельзя).
                        ; Релиз этой версии обязан иметь тег v1.2.0
type     = content      ; content | graphics | audio | script | ui | localization | library | location
api_min  = 0.1.0        ; минимальная версия API аддонов, под которую сделан аддон (см. "Совместимость").
                        ; Не задан - предупреждение, в лаунчере "совместимость не указана"
game_version = v1.4.0   ; тег игры, на которой сделаны ЗАМЕНЫ файлов базы целиком (см. ниже).
                        ; Аддону без таких замен не нужен
depends  = gw_npc, my_lib >=1.2.0   ; без этих аддонов наш не работает; условие версии - необязательно
; load_after = other_addon          ; мягкий порядок: встать после other_addon, если он установлен

; Опасные возможности Lua-кода. Сканер каталога ищет их в скриптах пакета.
; Используется, но не объявлено - ошибка caps_undeclared; новое по сравнению с прошлой версией - ручная проверка.
[capabilities]
fs_write                ; запись файлов: io.open(..., "w"/"a"/"+"), io.output, os.remove/rename/tmpname, lfs.mkdir ...
; ffi                   ; нативный код: ffi.*, require "ffi", package.loadlib, package.cpath
; os_exec               ; запуск программ: os.execute, io.popen

[lua]
entry = my_weapons_main ; файл gamedata\scripts\my_weapons_main.script в пакете с функцией on_load(ctx).
                        ; Имя начинается с "<id>_". Нет такого файла в пакете - ошибка lua_entry_missing
```

Секции `[capabilities]` и `[lua]` нужны только аддонам со скриптами. Ключ без значения (или `= true`) объявляет
возможность.

Кроме трёх возможностей, сканер отмечает приёмы, которые прячут вызовы: `load`/`loadstring`/`loadfile`/`dofile` не от
строкового литерала, `string.dump`, библиотека `debug` (кроме `traceback`, `getinfo`), `io.open` с режимом не
литералом, `os`/`io`/`ffi`/... как значение или через `[...]`. Это не ошибка, но такую версию смотрит модератор.

**Зачем `game_version`.** Если аддон кладёт в `gamedata\` целый файл базы игры (`.ltx`, `.xml` или `.script` без
префикса `mod_`), игра при установке сливает его с текущей базой, чтобы не откатить правки, вышедшие в игре позже.
Для этого CI каталога считает дельту «что аддон поменял относительно базы» и хранит её рядом с версией
(`files/<id>/<ver>.merge.json` в индексе). Базу CI берёт по `game_version`. Без поля версия базы угадывается (последний
релиз игры на дату публикации), и при ошибке слияние может откатить чужие правки — валидатор предупреждает
(`game_version_missing`). Лучше переписать такие правки на DLTX (`mod_*.ltx`) или хуки, тогда замен нет совсем.

## 2. Заполнить `catalog/`

`catalog/catalog.ltx` — тот же простой LTX (UTF-8 без BOM), но с одним отличием от `addon.ltx`: `//` внутри ссылок
**не** комментарий. Комментарий здесь — `;` или ` //` после пробела, поэтому `https://...` пишется как есть.

```ini
[catalog]
name        = Vector weapons pack            ; название без языка: показывается, если нет нужного перевода
name_ru     = Оружейный пак «Вектор»         ; перевод: name_<язык> (сейчас в лаунчере ru, en)
summary     = 12 new guns sold by traders    ; одна строка для списка аддонов, до 200 символов
summary_ru  = 12 новых стволов у торговцев
tags        = weapons, traders               ; через запятую, до 10, ТОЛЬКО из tags.ltx репо index
authors     = Иван Петров, Some Modder       ; авторы оригинала: свободный текст через запятую
maintainers = ivan-petrov                    ; GitHub-логины тех, кто публикует релизы
license     = MIT                            ; SPDX-идентификатор или custom; не задано - CC-BY-NC-4.0
permission  = https://example.com/post/123   ; ссылка на разрешение автора, если публикует не автор
source      = https://github.com/me/weapons  ; где исходники разработки (необязательно)
homepage    = https://example.com/weapons    ; необязательно
donate      = https://boosty.to/me           ; необязательно
```

- Обязательно хотя бы одно из `name` / `name_<язык>` и хотя бы одно из `summary` / `summary_<язык>`. Название — до
  80 символов.
- Язык выбирается по локали лаунчера: сначала `<поле>_<язык>`, потом поле без суффикса, потом запасная локаль
  лаунчера (`ru`), потом любой имеющийся вариант. Так же выбирается `description*.md`.
- **Описание** — `description.md` (без языка) и/или `description.ru.md`, `description.en.md`; нужен хотя бы один
  файл, каждый до 64 КБ, UTF-8.
- **Иконка** — `icon.png`, ровно 256x256, не больше 256 КБ.
- **Скриншоты** — `screenshots/`, до 10 файлов `.png`, `.jpg`/`.jpeg`, `.webp`, до 1920 px по длинной стороне, до
  1 МБ каждый, без подпапок.
- **Теги** — только ключи из `tags.ltx` репо `index` каталога (`weapons`, `armor`, `traders`, `npc`, `quests`,
  `locations`, `graphics`, `sounds`, `ui`, `gameplay`, ...). Чужой тег — ошибка `catalog_unknown_tag`.
- Изменения `catalog/` публикуются без модерации; ошибка в карточке — каталог оставляет прошлую карточку.

## 3. Проверить и упаковать локально

Валидатор — тот же, что запускает CI индекса. Он лежит в репо `index` каталога: `tools/validator/gw_addon_tool.py` —
инструмент автора (`validate`, `validate-repo`, `pack`, `remap-textures`), `tools/validator/gw_catalog.py` — те же
команды и команды CI. Рядом нужен `tools/addons_snapshot` — он там же. Нужен Python 3.10+ (для zip с zstd — 3.14+),
сторонние пакеты не нужны.

`gw_addon_tool.py` сам берёт `--index` — репо `index`, в котором лежит (теги, зарезервированные id, адрес снимков
базы). Со снимком базы проверка такая же, как в CI: какие файлы базы заменяет аддон, ссылки моделей на текстуры и
анимации, дельта текстов. Снимок валидатор скачивает сам по `source.ltx [game] snapshot_url` (в папку временных
файлов), или укажите папку со снимками: `--snapshots <папка> --latest <тег>` (`<папка>/<тег>/base_snapshot.json.gz`
и `base_texts.tar.gz`, их собирает `tools/addons_snapshot/gw_snapshot.py build`).

```bash
# Один раз: склонировать индекс рядом (папка tools/ + tags.ltx, reserved_ids.ltx нужны для проверки тегов и id).
git clone https://github.com/Global-War-Addons/index ../index

# 1. Этот репозиторий: addon.ltx + catalog/. --id - имя репо (оно же id аддона).
#    Код выхода 0 - всё хорошо, 1 - есть ошибки (отчёт их перечисляет).
python ../index/tools/validator/gw_addon_tool.py validate-repo . --id my_weapons

# 2. Рабочая папка аддона (addon.ltx + gamedata/ ...): раскладка пакета, запрещённые файлы, сканер Lua, depends,
#    замены файлов базы (раздел 3а). Папка с .git внутри - нормально: .git в пакет не попадает.
python ../index/tools/validator/gw_addon_tool.py validate D:/mods/my_weapons

# 3. Упаковать: сначала та же проверка, потом dist/my_weapons-1.2.0.zip.
#    Больше 2 ГиБ - сам режет на my_weapons-1.2.0.part1.zip, part2.zip ... (addon.ltx - в part1).
#    --out указывайте ВНЕ папки аддона, иначе при следующей упаковке dist/ попадёт в пакет.
python ../index/tools/validator/gw_addon_tool.py pack D:/mods/my_weapons --out dist

# 4. Проверить готовые архивы так же, как это сделает release.yml и CI индекса.
python ../index/tools/validator/gw_addon_tool.py validate --assets-dir dist --id my_weapons --version 1.2.0
```

`pack` запускается на рабочей папке аддона, **не на этом репозитории**: `catalog/` и `.github/` в пакете — ошибки.

**Что можно положить в пакет** (корень zip = корень папки аддона): `addon.ltx`, `gamedata/`, `merge/` (только у патча
совместимости с `[addon] patches`), папки файлов слотов `[slot.*]`, `README*`, `LICENSE*`, `CHANGELOG*`.

**Что нельзя:** `bin/` (сборки плагинов приходят только от CI, в v1 их нет), исполняемые файлы и скрипты ОС
(`.exe .dll .bat .cmd .ps1 .sh .vbs .so .dylib .com .scr .msi`), `<id>.db` (в v1 только папки), `catalog/`, пути с
`..`, абсолютные пути, символьные ссылки, `\` в именах внутри zip, шифрование. Сжатие — stored, deflate или zstd;
каждая часть меньше 2 ГиБ.

## 3а. Замены файлов базы, `[private]` и `remap-textures`

Файлы `gamedata\` аддона получают в игре те же имена, что у базы. Файл с путём базового **заменяет** его для всех —
как ручная правка gamedata. Тексты (`.ltx`, `.xml`, `.script` без префикса `mod_`) целиком не заменяются: игра сливает
правки аддона с текущей базой (см. `game_version` в разделе 1).

- **Что заменяет аддон, видно всем.** Валидатор пишет список в отчёт (`replaces_base`), а CI — в карточку каталога:
  лаунчер показывает «заменяет базовые файлы: N» и список (`files/<id>/<ver>.json`, поле `replaces`). В игре — строка
  лога `~ [addon:<id>] replaces N base files` и консольная команда `addon_files <id>`; `addon_resource <имя>`
  показывает, из какого аддона загружен ресурс.
- **Свои имена аддона — `@<id>`.** Всё из `textures\` и `meshes\` аддона доступно ещё и как `textures\@<id>\...`,
  `meshes\@<id>\...`. Модели аддона сначала ищут текстуры, `.thm` и анимации там, поэтому старая модель с именами
  текстур базы показывает текстуры аддона. Папку с именем `@...` внутри аддона не создавайте: движок её не монтирует
  (`package_alias_folder`).
- **`[private]` — не заменять базу.** Маска на строку (ключ без `=`), путь от `gamedata\`, `*` — любые символы
  (включая `\`), `?` — один символ, регистр не важен. Такой файл базу не трогает: его видят только модели аддона и
  явные ссылки `@<id>\...`. Чужие ресурсы `@<другой_id>\...` — только при `depends` (`ref_without_depends`).

```ini
[private]
textures\wpn\wpn_ak74.dds     ; своя текстура для своей модели, базовый АК остаётся прежним
textures\wpn\wpn_ak74.thm
textures\my_weapons\*         ; все текстуры своей папки
```

Конфиги, UI xml и скрипты ищут имена глобально: на приватную текстуру из них ссылайтесь как `@my_weapons\wpn\x`.

**`remap-textures`** делает это за вас: текстуры аддона с именами базы (по снимку базы `game_version`) уходят в
`[private]` вместе с их `.thm`, а в моделях аддона (`gamedata\meshes\**\*.ogf`, наборы слотов) имена текстур
переписываются на `@<id>\<имя>`. Файлы меняются на месте — сначала закоммитьте папку или сделайте копию.

```bash
# что изменится, ничего не записывая
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --dry-run
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons

# снимок базы: по умолчанию тег [addon] game_version (иначе последний релиз); или явно
#   --snapshots <папка> --game v1.4.0   |   --base-snapshot <файл base_snapshot.json.gz>
# все текстуры аддона, не только с именами базы (снимок не нужен), и только часть по маске
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --all --only "textures\my_weapons\*"
# только [private] без правки моделей / только модели без [private]
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --no-models
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --no-private
```

- Текстура, имя которой встречается в конфигах, UI xml или скриптах аддона, остаётся публичной (предупреждение
  `remap_text_reference`): перепишите эти ссылки на `@<id>\<имя>` и запустите снова.
- `.thm` не переписывается: bump и detail для `@<id>\...` движок находит сам. Без своего `.thm` приватная текстура
  берёт `.thm` базы с тем же именем; приватная текстура с `_bump` без `.thm` теряет bump (`private_texture_no_thm`).
- Не обрабатываются: сжатые чанки OGF, геометрия уровней (`levels\`), ссылки в текстах (см. выше).
- Повторный запуск ничего не меняет. Модель с именами `@<id>\...` в другом аддоне потребует его в `depends`.

Предупреждения о заменах и ресурсах (публикацию не останавливают):

| Код | Что значит |
|---|---|
| `replaces_save_breaking` | заменяет `spawns\all.spawn` или `levels\*\level.ai/.game/.spawn`: сохранения без аддона с ним не загрузятся (новый мир — `type = location`) |
| `replaces_single_file_base` | заменяет `particles.xr`, `shaders.xr` или `lanims.xr` целиком: в v1 это дело `total_conversion` |
| `replaces_root_config` | `configs\system.ltx` или `game.ltx` целиком: лучше DLTX `mod_system_<id>.ltx` |
| `replaces_unsupported` | `gamemtl.xr`, `configs\misc\slots\*`: игра читает их до подключения аддонов, файл не действует |
| `replaces_texture_own_model` | заменённую текстуру базы используют новые модели аддона — похоже на случайное совпадение имени: `[private]` или `remap-textures` |
| `private_mask_unused` | маска `[private]` не нашла ни одного файла |
| `model_texture_missing`, `model_motion_missing`, `model_lod_missing` | модель ссылается на текстуру, анимацию или LOD, которых нет ни в аддоне, ни в базе (при `depends` — заметка) |

## 4. Опубликовать версию

1. Обновить `version` в `addon.ltx` рабочей папки, упаковать (`pack`) и скопировать тот же `addon.ltx` в этот репо.
2. Создать Release с тегом **`v<version>`** — ровно `[addon] version` из пакета (`v1.2.0` для `1.2.0`).
3. Прикрепить `dist/*.zip`: `<id>-<ver>.zip` или все части `<id>-<ver>.part<N>.zip` (не то и другое сразу).
4. **Заметки релиза** = changelog версии (markdown), лаунчер показывает их при обновлении. Если первая строка
   содержит `breaks-saves`, версия помечается как ломающая сохранения (автообновление её пропускает).
5. **Pre-release** — бета-канал: версию видят только игроки с включёнными бета-версиями. **Draft** CI каталога не
   видит вовсе.

Через веб GitHub: «Releases» → «Draft a new release» → тег `v1.2.0` → заметки → перетащить zip → «Publish release».

Через GitHub CLI:

```bash
# Тег создаётся на текущем коммите. Первая строка CHANGELOG.md попадает в первую строку заметок:
# напишите в ней "breaks-saves", если версия ломает сохранения.
gh release create v1.2.0 dist/*.zip --title "1.2.0" --notes-file CHANGELOG.md

# Бета-версия: тот же релиз с флагом --prerelease.
gh release create v1.3.0 dist/*.zip --prerelease --notes-file CHANGELOG.md
```

**Ассеты опубликованной версии не заменять.** Если CI уже принял версию (она в каталоге или на модерации), а ассет
потом заменили, CI отзывает версию (`yanked`) и открывает Issue модераторам. Исправление — новая версия
(`1.2.1`, тег `v1.2.1`). Исключение — версия, которую валидатор отклонил: исправленный пакет можно залить в тот же
релиз заново.

## 5. Что происходит дальше

1. **`release.yml` этого репо** (необязательный) скачивает ассеты релиза и проверяет их валидатором; результат — в
   сводке запуска Actions. Если у организации настроено приложение каталога (`vars.CATALOG_DISPATCH_APP_ID`), он
   просит индекс проверить релиз сразу; иначе индекс найдёт релиз сам при следующем запуске по расписанию (раз в
   30 минут). Опубликовать версию этот workflow не может.
2. **CI индекса** скачивает ассеты, считает sha256 и проверяет всё заново тем же валидатором.
3. Версию проверяет **модератор** (статус `pending`, Issue «Review» в репо `index`), если это первая публикация
   аддона, расширились `[capabilities]`, сработал сканер Lua или сменился `type`.
4. Иначе версия попадает в каталог **автоматически**.
5. **Ошибки валидатора** — версия не публикуется, в этом репо появляется Issue с отчётом.

## 6. Совместимость: `api_min`

Совместимость определяется не версией игры, а «мажором» версии API аддонов. Для `1.x.y` и выше мажор — первое число,
для `0.x.y` — пара `0.x` (до 1.0 ломающим считается рост второго числа). Аддон совместим, если мажор `api_min` совпадает
с мажором движка и `api_min` не новее версии движка.

| API движка | `api_min` аддона | Итог |
|---|---|---|
| `0.1.3` | `0.1.0` | совместим |
| `0.1.3` | `0.1.5` | нужна более новая версия игры |
| `0.2.0` | `0.1.0` | несовместим, нужно обновить аддон |
| `1.4.0` | `1.0.0` | совместим |
| `2.0.0` | `1.4.0` | несовместим, нужно обновить аддон |
| любая | не задан | грузится, «совместимость не указана» |

Каталог по умолчанию показывает только версии под мажор установленной игры, поэтому после ломающего обновления API
выпустите новую версию аддона с новым `api_min`. Ставьте `api_min` равным версии API, на которой проверяли аддон.

## 7. Частые ошибки валидатора

| Код | Что не так | Как исправить |
|---|---|---|
| `repo_placeholder` | в `addon.ltx` или `catalog.ltx` остались `{{...}}` | заполнить поля шаблона |
| `manifest_id_mismatch` | `[addon] id` не равен имени репо | `id` = имя репо |
| `manifest_version_mismatch` | `version` в пакете не равен тегу | тег `v<version>` или поправить `version` и перепаковать |
| `asset_no_package` | у релиза нет `<id>-<ver>.zip` / `.part1.zip` | прикрепить пакет с точным именем (так его называет `pack`) |
| `asset_whole_and_parts` | у релиза и `.zip`, и `.part<N>.zip` | оставить что-то одно |
| `package_nested` | `addon.ltx` лежит в подпапке zip | корень zip = корень папки аддона; паковать через `pack` |
| `package_unknown_folder` | в корне пакета лишняя папка (`textures/`, `.github/`, `dist/`) | всё содержимое — в `gamedata/` |
| `package_catalog` | в пакете `catalog/` | паковать рабочую папку аддона, а не этот репо |
| `package_executable` | в пакете `.exe`, `.dll`, `.bat`, `.sh` ... | убрать их из пакета |
| `package_bin` | в пакете `bin/` | убрать: плагины из каталога в v1 не принимаются |
| `package_db_v1` | в корне пакета `<id>.db` | распаковать архив в `gamedata/` |
| `package_name_encoding` | не-ASCII имена без флага UTF-8 в zip | паковать через `pack` или 7-Zip |
| `caps_undeclared` | скрипт использует `fs_write` / `ffi` / `os_exec` без объявления | добавить ключ в `[capabilities]` или убрать вызов |
| `lua_entry_missing` | нет `gamedata/scripts/<entry>.script` для `[lua] entry` | положить скрипт или исправить `entry` |
| `ref_without_depends` | файлы ссылаются на `@другой_аддон\...`, а его нет в `depends` | добавить его в `[addon] depends` |
| `catalog_no_description` | в `catalog/` нет ни одного `description*.md` | добавить `description.md` |
| `catalog_icon_size` | `icon.png` не 256x256 | пересохранить иконку 256x256 |
| `catalog_unknown_tag` | тега нет в `tags.ltx` индекса | взять тег из `tags.ltx` |

Предупреждения (`manifest_no_api_min`, `game_version_missing`, `namespace_*`) публикацию не останавливают, но их стоит
исправить. Полный список кодов — в отчёте валидатора и в исходниках `tools/validator/gwcat/`.
