Русская версия: [README.md](README.md)

# GlobalWar addon repository template

> **In short.** This repo is the catalog entry of one GlobalWar addon: `addon.ltx` (the manifest) and
> `catalog/` (the card shown in the launcher). The addon files themselves are **not** in git: every version is a
> GitHub Release with the package `<id>-<version>.zip` attached as an asset. Fill `addon.ltx` and `catalog/`,
> check them with `gw_addon_tool.py validate-repo`, pack the addon with `gw_addon_tool.py pack`, publish a release tagged
> `v<version>`. The central CI of the index validates it again and publishes it (or opens an Issue with the errors).

## What this repository is

One repository = one catalog addon. The repository name equals the addon's `id`. The repository contains only:

- `addon.ltx` — the addon manifest (the same file as in the package of the latest version);
- `catalog/` — the addon's card in the launcher: name, description, icon, screenshots.

**The addon files (`gamedata/` etc.) are not committed to git.** Each version is a GitHub Release with a zip package
attached. This way GitHub's repository size limits don't get in the way of large packs: the only limit is on a single
release file (under 2 GiB), and a large package is split into parts.

Development sources (the addon working folder, source textures, SDK projects, Blender files) can live anywhere: in
another repository, on disk, in the cloud. The launcher doesn't read them.

## Layout

| Path | Required | What it is |
|---|---|---|
| `addon.ltx` | yes | manifest; `id` = repo name |
| `catalog/catalog.ltx` | yes | card fields: name, short description, tags, authors, license, links |
| `catalog/description.md`, `catalog/description.<lang>.md` | at least one | addon description (markdown) |
| `catalog/icon.png` | no | 256x256 icon |
| `catalog/screenshots/` | no | up to 10 screenshots |
| `.github/workflows/validate.yml` | no | checks `addon.ltx` and `catalog/` on push |
| `.github/workflows/release.yml` | no | checks the assets when a release is published |
| `README.md`, `LICENSE` | no | for GitHub; the launcher doesn't read them |

## 1. Fill in `addon.ltx`

If the repo was created by the catalog bot, the `{{...}}` placeholders are already filled in. If you created the repo
yourself, fill them in: any leftover `{{...}}` is a `repo_placeholder` error. All manifest fields are described in the
main project's wiki: `wiki/doc/plugins/manifest.md`.

```ini
; A comment in addon.ltx runs from ";" or "//" to the end of the line. The engine cuts at EVERY "//",
; so don't put links (https://...) in addon.ltx - catalog/catalog.ltx is the place for them.

[addon]
id       = my_weapons   ; = repo name: only a-z 0-9 _, up to 64 characters. Otherwise manifest_id_mismatch
version  = 1.2.0        ; three numbers MAJOR.MINOR.PATCH, no suffixes (-beta is not allowed).
                        ; The release of this version must have the tag v1.2.0
type     = content      ; content | graphics | audio | script | ui | localization | library | location
api_min  = 0.1.0        ; minimum Addons API version the addon is made for (see "Compatibility").
                        ; Not set - a warning, the launcher shows "Compatibility not stated"
game_version = v1.4.0   ; game tag the WHOLE-FILE replacements of base files were made against (see below).
                        ; Not needed for an addon without such replacements
depends  = gw_npc, my_lib >=1.2.0   ; our addon doesn't work without these; the version condition is optional
; load_after = other_addon          ; soft ordering: load after other_addon if it is installed

; Dangerous Lua code capabilities. The catalog scanner looks for them in the package scripts.
; Used but not declared - caps_undeclared error; new compared to the previous version - manual review.
[capabilities]
fs_write                ; writing files: io.open(..., "w"/"a"/"+"), io.output, os.remove/rename/tmpname, lfs.mkdir ...
; ffi                   ; native code: ffi.*, require "ffi", package.loadlib, package.cpath
; os_exec               ; running programs: os.execute, io.popen

[lua]
entry = my_weapons_main ; the file gamedata\scripts\my_weapons_main.script in the package with an on_load(ctx) function.
                        ; The name starts with "<id>_". No such file in the package - lua_entry_missing error
```

The `[capabilities]` and `[lua]` sections are only needed for addons with scripts. A key without a value (or `= true`)
declares a capability.

Besides the three capabilities, the scanner flags techniques that hide calls: `load`/`loadstring`/`loadfile`/`dofile`
on anything other than a string literal, `string.dump`, the `debug` library (except `traceback`, `getinfo`), `io.open`
with a non-literal mode, `os`/`io`/`ffi`/... used as a value or accessed via `[...]`. This is not an error, but a
moderator reviews such a version.

**Why `game_version`.** If an addon puts a whole base game file into `gamedata\` (`.ltx`, `.xml` or `.script` without
the `mod_` prefix), the game merges it with the current base on install, so that changes released in the game later
are not rolled back. For this, the catalog CI computes a delta of "what the addon changed relative to the base" and
stores it next to the version (`files/<id>/<ver>.merge.json` in the index). CI picks the base by `game_version`. Without
the field, the base version is guessed (the latest game release as of the publication date), and if the guess is wrong
the merge may roll back other people's changes — the validator warns about this (`game_version_missing`). It's better to
rewrite such changes as DLTX (`mod_*.ltx`) or hooks; then there are no replacements at all.

## 2. Fill in `catalog/`

`catalog/catalog.ltx` is the same simple LTX (UTF-8 without BOM), with one difference from `addon.ltx`: `//` inside
links is **not** a comment. A comment here is `;` or ` //` preceded by a space, so `https://...` is written as is.

```ini
[catalog]
name        = Vector weapons pack            ; name without a language: shown when there is no matching translation
name_ru     = Оружейный пак «Вектор»         ; translation: name_<lang> (the launcher currently has ru, en)
summary     = 12 new guns sold by traders    ; one line for the addon list, up to 200 characters
summary_ru  = 12 новых стволов у торговцев
tags        = weapons, traders               ; comma-separated, up to 10, ONLY from tags.ltx of the index repo
authors     = Иван Петров, Some Modder       ; authors of the original: free text, comma-separated
maintainers = ivan-petrov                    ; GitHub logins of the people who publish releases
license     = MIT                            ; SPDX identifier or custom; not set - CC-BY-NC-4.0
permission  = https://example.com/post/123   ; link to the author's permission, if the publisher is not the author
source      = https://github.com/me/weapons  ; where the development sources are (optional)
homepage    = https://example.com/weapons    ; optional
donate      = https://boosty.to/me           ; optional
```

- At least one of `name` / `name_<lang>` and at least one of `summary` / `summary_<lang>` are required. The name is up
  to 80 characters.
- The language is picked by the launcher locale: first `<field>_<lang>`, then the field without a suffix, then the
  launcher's fallback locale (`ru`), then any available variant. `description*.md` is picked the same way.
- **Description** — `description.md` (no language) and/or `description.ru.md`, `description.en.md`; at least one file
  is required, each up to 64 KB, UTF-8.
- **Icon** — `icon.png`, exactly 256x256, no larger than 256 KB.
- **Screenshots** — `screenshots/`, up to 10 `.png`, `.jpg`/`.jpeg`, `.webp` files, up to 1920 px on the long side, up
  to 1 MB each, no subfolders.
- **Tags** — only keys from `tags.ltx` in the catalog's `index` repo (`weapons`, `armor`, `traders`, `npc`, `quests`,
  `locations`, `graphics`, `sounds`, `ui`, `gameplay`, ...). An unknown tag is a `catalog_unknown_tag` error.
- Changes to `catalog/` are published without moderation; if the card has an error, the catalog keeps the previous card.

## 3. Validate and pack locally

The validator is the same one the index CI runs. It lives in the catalog's `index` repo: `tools/validator/gw_addon_tool.py`
is the author tool (`validate`, `validate-repo`, `pack`, `remap-textures`), `tools/validator/gw_catalog.py` has the same
commands plus the CI commands. They also need `tools/addons_snapshot` next to them, which is in the same repo. Python
3.10+ is required (3.14+ for zip with zstd); no third-party packages are needed.

`gw_addon_tool.py` sets `--index` on its own to the `index` repo it lives in (tags, reserved ids, the base snapshot
URL). With a base snapshot, the check is the same as in CI: which base files the addon replaces, model references to
textures and animations, the text delta. The validator downloads the snapshot itself from `source.ltx [game] snapshot_url`
(into the temp folder), or you can point it to a folder with snapshots: `--snapshots <folder> --latest <tag>`
(`<folder>/<tag>/base_snapshot.json.gz` and `base_texts.tar.gz`, built by `tools/addons_snapshot/gw_snapshot.py build`).

```bash
# Once: clone the index next to this repo (the tools/ folder + tags.ltx, reserved_ids.ltx are needed to check tags and ids).
git clone https://github.com/Global-War-Addons/index ../index

# 1. This repository: addon.ltx + catalog/. --id - the repo name (which is also the addon id).
#    Exit code 0 - all good, 1 - there are errors (the report lists them).
python ../index/tools/validator/gw_addon_tool.py validate-repo . --id my_weapons

# 2. The addon working folder (addon.ltx + gamedata/ ...): package layout, forbidden files, Lua scanner, depends,
#    base file replacements (section 3a). A folder with .git inside is fine: .git does not go into the package.
python ../index/tools/validator/gw_addon_tool.py validate D:/mods/my_weapons

# 3. Pack: first the same check, then dist/my_weapons-1.2.0.zip.
#    Over 2 GiB - it splits the package into my_weapons-1.2.0.part1.zip, part2.zip ... by itself (addon.ltx goes into part1).
#    Put --out OUTSIDE the addon folder, otherwise dist/ ends up in the package the next time you pack.
python ../index/tools/validator/gw_addon_tool.py pack D:/mods/my_weapons --out dist

# 4. Check the finished archives the same way release.yml and the index CI will.
python ../index/tools/validator/gw_addon_tool.py validate --assets-dir dist --id my_weapons --version 1.2.0
```

`pack` runs on the addon working folder, **not on this repository**: `catalog/` and `.github/` in the package are
errors.

**What may go into the package** (zip root = addon folder root): `addon.ltx`, `gamedata/`, `merge/` (only for a
compatibility patch with `[addon] patches`), folders with `[slot.*]` slot files, `README*`, `LICENSE*`, `CHANGELOG*`.

**What may not:** `bin/` (plugin builds come only from CI, and v1 has none), OS executables and scripts
(`.exe .dll .bat .cmd .ps1 .sh .vbs .so .dylib .com .scr .msi`), `<id>.db` (v1 accepts folders only), `catalog/`, paths
with `..`, absolute paths, symbolic links, `\` in names inside the zip, encryption. Compression: stored, deflate or
zstd; each part under 2 GiB.

## 3a. Base file replacements, `[private]` and `remap-textures`

Files in the addon's `gamedata\` get the same names in the game as the base files. A file with a base file's path
**replaces** it for everyone — just like a manual gamedata edit. Text files (`.ltx`, `.xml`, `.script` without the
`mod_` prefix) are not replaced wholesale: the game merges the addon's changes with the current base (see
`game_version` in section 1).

- **What an addon replaces is visible to everyone.** The validator writes the list to its report (`replaces_base`), and
  CI writes it to the catalog card: the launcher shows "replaces game files: N" and the list (`files/<id>/<ver>.json`,
  field `replaces`). In the game there is the log line `~ [addon:<id>] replaces N base files` and the console command
  `addon_files <id>`; `addon_resource <name>` shows which addon a resource was loaded from.
- **The addon's own names: `@<id>`.** Everything in the addon's `textures\` and `meshes\` is also available as
  `textures\@<id>\...`, `meshes\@<id>\...`. The addon's models look for textures, `.thm` and animations there first, so
  an old model that uses base texture names shows the addon's textures. Don't create a folder named `@...` inside the
  addon: the engine doesn't mount it (`package_alias_folder`).
- **`[private]` — don't replace the base.** One mask per line (a key without `=`), path relative to `gamedata\`, `*` —
  any characters (including `\`), `?` — one character, case-insensitive. Such a file doesn't touch the base: only the
  addon's models and explicit `@<id>\...` references see it. Another addon's resources `@<other_id>\...` — only with
  `depends` (`ref_without_depends`).

```ini
[private]
textures\wpn\wpn_ak74.dds     ; own texture for own model, the base AK stays unchanged
textures\wpn\wpn_ak74.thm
textures\my_weapons\*         ; all textures in the addon's own folder
```

Configs, UI xml and scripts look names up globally: from them, refer to a private texture as `@my_weapons\wpn\x`.

**`remap-textures`** does this for you: addon textures with base names (according to the base snapshot for
`game_version`) go into `[private]` together with their `.thm`, and in the addon's models (`gamedata\meshes\**\*.ogf`,
slot sets) texture names are rewritten to `@<id>\<name>`. Files are changed in place — commit the folder or make a copy
first.

```bash
# what will change, without writing anything
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --dry-run
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons

# base snapshot: by default the [addon] game_version tag (otherwise the latest release); or explicitly
#   --snapshots <folder> --game v1.4.0   |   --base-snapshot <base_snapshot.json.gz file>
# all addon textures, not only those with base names (no snapshot needed), and only a subset by mask
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --all --only "textures\my_weapons\*"
# only [private] without editing models / only models without [private]
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --no-models
python ../index/tools/validator/gw_addon_tool.py remap-textures D:/mods/my_weapons --no-private
```

- A texture whose name appears in the addon's configs, UI xml or scripts stays public (warning
  `remap_text_reference`): rewrite those references to `@<id>\<name>` and run it again.
- `.thm` is not rewritten: the engine finds bump and detail for `@<id>\...` on its own. Without its own `.thm`, a
  private texture takes the base `.thm` with the same name; a private texture with `_bump` and no `.thm` loses its
  bump (`private_texture_no_thm`).
- Not handled: compressed OGF chunks, level geometry (`levels\`), references in text files (see above).
- Running it again changes nothing. A model in another addon that uses `@<id>\...` names will require this addon in
  its `depends`.

Warnings about replacements and resources (they don't block publishing):

| Code | Meaning |
|---|---|
| `replaces_save_breaking` | replaces `spawns\all.spawn` or `levels\*\level.ai/.game/.spawn`: saves made without the addon won't load with it (a new world is `type = location`) |
| `replaces_single_file_base` | replaces `particles.xr`, `shaders.xr` or `lanims.xr` wholesale: in v1 that's a job for `total_conversion` |
| `replaces_root_config` | `configs\system.ltx` or `game.ltx` wholesale: DLTX `mod_system_<id>.ltx` is better |
| `replaces_unsupported` | `gamemtl.xr`, `configs\misc\slots\*`: the game reads them before addons are mounted, so the file has no effect |
| `replaces_texture_own_model` | a replaced base texture is used by the addon's new models — looks like an accidental name clash: use `[private]` or `remap-textures` |
| `private_mask_unused` | a `[private]` mask matched no files |
| `model_texture_missing`, `model_motion_missing`, `model_lod_missing` | a model references a texture, animation or LOD that is neither in the addon nor in the base (with `depends` — a note) |

## 4. Publish a version

1. Update `version` in the working folder's `addon.ltx`, pack it (`pack`) and copy the same `addon.ltx` into this repo.
2. Create a Release tagged **`v<version>`** — exactly `[addon] version` from the package (`v1.2.0` for `1.2.0`).
3. Attach `dist/*.zip`: `<id>-<ver>.zip` or all the parts `<id>-<ver>.part<N>.zip` (not both).
4. **Release notes** = the version's changelog (markdown); the launcher shows them on update. If the first line
   contains `breaks-saves`, the version is marked as save-breaking (auto-update skips it).
5. **Pre-release** is the beta channel: only players with beta versions enabled see the version. A **Draft** is not
   visible to the catalog CI at all.

Via the GitHub website: "Releases" → "Draft a new release" → tag `v1.2.0` → notes → drag in the zip → "Publish release".

Via the GitHub CLI:

```bash
# The tag is created on the current commit. The first line of CHANGELOG.md becomes the first line of the notes:
# write "breaks-saves" there if the version breaks saves.
gh release create v1.2.0 dist/*.zip --title "1.2.0" --notes-file CHANGELOG.md

# Beta version: the same release with the --prerelease flag.
gh release create v1.3.0 dist/*.zip --prerelease --notes-file CHANGELOG.md
```

From the GlobalWar launcher (no Python or `gh` needed): the "Addons" view → **"Author mode"**. The launcher checks the
working folder with the same rules as the validator (except the Lua scanner, references and base replacements — CI
checks those), packs it into `<id>-<ver>.zip` or 2 GiB parts (deflate by default; for zstd the local validator needs
Python 3.14+), creates the `v<version>` release as a **draft**, uploads and verifies every part (size, sha256) and only
then publishes it — CI never sees an incomplete release. Running it again resumes the upload. It needs a fine-grained
GitHub token with **Contents: Read and write** on this repository only; a remembered token is stored under DPAPI on
Windows (on Linux/macOS it is only obfuscated). The launcher never changes a published release on its own: its assets
can be replaced only with the explicit "Allow changing a release already published" checkbox and a confirmation (see below
why you shouldn't). Details: the project wiki, section "Addons", page "Publishing to the catalog"
(`doc/addons/publishing.md`).

**Don't replace the assets of a published version.** If CI has already accepted a version (it is in the catalog or
under moderation) and an asset is replaced afterwards, CI yanks the version (`yanked`) and opens an Issue for the
moderators. The fix is a new version (`1.2.1`, tag `v1.2.1`). The exception is a version the validator rejected: you
can upload the fixed package to the same release again.

## 5. What happens next

1. **This repo's `release.yml`** (optional) downloads the release assets and checks them with the validator; the result
   is in the Actions run summary. If the organization has the catalog app set up (`vars.CATALOG_DISPATCH_APP_ID`), it
   asks the index to check the release right away; otherwise the index finds the release on its own on the next
   scheduled run (every 30 minutes). This workflow cannot publish a version.
2. **The index CI** downloads the assets, computes sha256 and re-checks everything with the same validator.
3. A **moderator** reviews the version (status `pending`, a "Review" Issue in the `index` repo) if this is the addon's
   first publication, `[capabilities]` were extended, the Lua scanner fired, or `type` changed.
4. Otherwise the version enters the catalog **automatically**.
5. **Validator errors** — the version is not published, and an Issue with the report appears in this repo.

## 6. Compatibility: `api_min`

Compatibility is determined not by the game version but by the "major" of the Addons API version. For `1.x.y` and
above the major is the first number; for `0.x.y` it is the pair `0.x` (before 1.0, a bump of the second number counts as
breaking). An addon is compatible if the major of `api_min` matches the engine's major and `api_min` is not newer than
the engine version.

| Engine API | Addon `api_min` | Result |
|---|---|---|
| `0.1.3` | `0.1.0` | compatible |
| `0.1.3` | `0.1.5` | a newer game version is needed |
| `0.2.0` | `0.1.0` | incompatible, the addon needs an update |
| `1.4.0` | `1.0.0` | compatible |
| `2.0.0` | `1.4.0` | incompatible, the addon needs an update |
| any | not set | loads, "Compatibility not stated" |

By default the catalog shows only versions for the installed game's major, so after a breaking API update, release a
new version of the addon with the new `api_min`. Set `api_min` to the API version you tested the addon against.

## 7. Common validator errors

| Code | What's wrong | How to fix |
|---|---|---|
| `repo_placeholder` | `{{...}}` left in `addon.ltx` or `catalog.ltx` | fill in the template fields |
| `manifest_id_mismatch` | `[addon] id` doesn't match the repo name | `id` = repo name |
| `manifest_version_mismatch` | `version` in the package doesn't match the tag | tag `v<version>`, or fix `version` and repack |
| `asset_no_package` | the release has no `<id>-<ver>.zip` / `.part1.zip` | attach the package with the exact name (the one `pack` gives it) |
| `asset_whole_and_parts` | the release has both `.zip` and `.part<N>.zip` | keep only one of them |
| `package_nested` | `addon.ltx` is in a subfolder of the zip | zip root = addon folder root; pack with `pack` |
| `package_unknown_folder` | an extra folder in the package root (`textures/`, `.github/`, `dist/`) | all content goes into `gamedata/` |
| `package_catalog` | `catalog/` in the package | pack the addon working folder, not this repo |
| `package_executable` | `.exe`, `.dll`, `.bat`, `.sh` ... in the package | remove them from the package |
| `package_bin` | `bin/` in the package | remove it: the catalog doesn't accept plugins in v1 |
| `package_db_v1` | `<id>.db` in the package root | unpack the archive into `gamedata/` |
| `package_name_encoding` | non-ASCII names without the UTF-8 flag in the zip | pack with `pack` or 7-Zip |
| `caps_undeclared` | a script uses `fs_write` / `ffi` / `os_exec` without declaring it | add the key to `[capabilities]` or remove the call |
| `lua_entry_missing` | no `gamedata/scripts/<entry>.script` for `[lua] entry` | add the script or fix `entry` |
| `ref_without_depends` | files reference `@other_addon\...`, but it is not in `depends` | add it to `[addon] depends` |
| `catalog_no_description` | no `description*.md` in `catalog/` | add `description.md` |
| `catalog_icon_size` | `icon.png` is not 256x256 | re-save the icon at 256x256 |
| `catalog_unknown_tag` | the tag is not in the index's `tags.ltx` | pick a tag from `tags.ltx` |

Warnings (`manifest_no_api_min`, `game_version_missing`, `namespace_*`) don't block publishing, but they are worth
fixing. The full list of codes is in the validator report and in the sources under `tools/validator/gwcat/`.
