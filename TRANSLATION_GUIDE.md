# Phantom of the Kill (offline build): translation hand-over guide

This guide explains how the in-game English translation works today, how to translate text and button images, and how to use the lists of
everything that is still untranslated. Everything below was written from the current repository state (`main`).

Two kinds of text exist, and they are translated in two different ways:

| Kind | Where it lives | How it is translated | Output file |
|---|---|---|---|
| **Text** (labels, messages, names, descriptions drawn by the game as text) | strings in code and master data | XUnity.AutoTranslator looks the Japanese string up in a dictionary | `translation/en.txt` |
| **Baked text** (buttons, banners, tabs: Japanese drawn inside a picture) | pixels inside NGUI atlas images | the picture is redrawn with English text and XUnity swaps the texture | `translation/texture_text.json` (+ `tools/texture_translate.py`) |

---

## 1. Folder layout and prerequisites

The tools assume the game folder is a sibling of the repository folder:

```
<work>/
  PoK/                 the game (PotK.exe, BepInEx/, Data/, PotK_Data/ ...)  - NOT part of the repository
  potk-offline/        this repository (tools/, translation/, server/ ...)
```

You need:
* Windows, PowerShell 5.1+, **Python 3.11+** with `pip install pillow numpy opencv-python UnityPy` (opencv/numpy are used by the button tool,
  UnityPy by the atlas tools).
* The game with BepInEx 5 + XUnity.AutoTranslator + the PotkWide plugin already installed (`PoK/BepInEx/`). `tools/i18n-sync.ps1` configures
  XUnity for offline use.
* The offline server (`play.bat` starts it together with the game; or run `server/PotkServer`). It needs the **.NET 10 SDK**. The server is
  what logs untranslated text (see 2.1), so run the game through `play.bat`.
* Optional (only if you regenerate lists): `data/master/json/` (master data dump) and `decompiled/` (client source). These are **not in
  git** because they are derived from the game; ask the maintainer or generate them with `tools/MasterDump`. The ready-made lists in
  `extracted/` already contain everything you need to start.

Never commit `PoK/`, `decompiled/`, `data/`, texture dumps or `translation/texture_orig/`: they contain the game's own assets/code.

---

## 2. Translating text (`translation/en.txt`)

### 2.1 How it works
1. The game shows a Japanese string; XUnity.AutoTranslator (BepInEx plugin) intercepts it.
2. XUnity first checks its static dictionary `PoK/BepInEx/Translation/en/Text/potk.txt` (a copy of `translation/en.txt`).
3. If it is not there, XUnity asks the local server: `GET http://127.0.0.1:5080/translate?text=...` (`server/PotkServer/Translate.cs`). The server
   matches exact text, then the trimmed text, then a normalised form (NGUI inserts line breaks, spaces and full-width `%` into long labels).
4. **Every miss is appended to `translation/untranslated.txt`.** That file is your to-do list of what players actually see.

### 2.2 File format
One entry per line: `japanese=english`.

```
ショップ=Shop
{0}体を獲得しました\n=Obtained {0} unit(s)\n
経験値=EXP
r:"^(\d+)回$"=$1x          // regex entry (use only when plain entries cannot work)
// this is a comment
```

Rules (checked by the linter):
* `\n` = line break, `\=` = a literal `=`. Everything else is literal.
* Placeholders (`{0}`, `%(name)s`, `$1`), colour/format tags (`<color=...>`, `[ffff00]...[-]`) and `\n` must be **identical** on both sides.
* No duplicate keys, no empty value, no raw tab.
* Never translate player-entered text (names, user codes, chat). Numbers stay numbers.
* Short UI labels (they must fit buttons): 1-2 words, Title Case. Plain polite English elsewhere, no slang.
* Follow `translation/glossary.md` (terms such as Rebirth, Merge, Summon, Lazuli, Zeny, Resonance). Add every new term to it (one line).
  Stat labels are `HP STR MAG DEF RES SPD SKL LUK`.
* Translate literally and closely (keep structure and roughly the same length). If unsure, translate literally and put `// ? reason` on the
  line above.
* Proper names of characters: use the established spelling if an entry/glossary already shows it; otherwise a plain romanisation marked
  with `// ?`.

Where to add lines: `translation/en.txt` is one big file. To avoid merge conflicts agree on one section per person and **append only**
(never reorder or reformat existing lines). Example: `// ---- <your name> entries ----` near the end of the file.

### 2.3 Daily loop
```powershell
# 1. get the current to-do list (writes to handover/extracted by default; pick any folder)
python -I tools/export_untranslated.py handover/extracted      # needs data/master/json for the master-data part
python -I tools/untranslated_names.py all > handover/extracted/unit_item_names_untranslated.tsv   # unit/item/gear/ticket names (needs data/master/json)
python -I tools/lint_translation.py --pending                  # lines of untranslated.txt that still have no entry

# 2. add lines to translation/en.txt, then check them
python -I tools/lint_translation.py                            # 0 errors required (warnings are OK, read them)

# 3. install into the game (CLOSE THE GAME FIRST: it also purges XUnity's cache of 'untranslated' lines)
powershell -ExecutionPolicy Bypass -File tools/i18n-sync.ps1

# 4. start the game with play.bat and look at the screens you changed (in game: ALT+R reloads the dictionary)
```
`tools/check.ps1` runs the server build, the lint and JSON checks in one go (the "definition of done" before you push).

### 2.4 Lists of untranslated text (`extracted/`)
| File | Content | How to use |
|---|---|---|
| `runtime_untranslated.txt` | text the game logged as untranslated (`translation/untranslated.txt` + `pending.txt`), ready as `japanese=` lines | **Highest priority.** Fill in the right side and paste into `en.txt`. |
| `consts_untranslated.tsv` | UI message constants of the client (`NAME<TAB>japanese`, `\n` = line break) | They appear as dialogs/toasts. Add `japanese=english` to `en.txt` (use the text column; keep `%(x)s`/`{0}`). |
| `unit_item_names_untranslated.tsv` | unit / material / gear / ticket names without English | add `japanese=english` lines. |
| `master/<Table>__<column>.tsv` | game-data strings (id, japanese) from **every** master-data column that contains Japanese (292 table/column files, about 131,000 strings, tables split into `_part_` files merged), e.g. `BattleskillSkill__name.tsv`, `UnitUnitDescription__description.tsv`, `ScriptScript__script.tsv` (story scripts) | **Large.** Only strings the game actually shows matter; work in the order of `master_summary.tsv`, or when a screen shows Japanese. One `en.txt` line fixes every copy of the same string. `ScriptScript` / `DateScriptBase` / `DateScriptQuestion` / `SeaHomeSerif` are story and date dialogue: an entry may contain several lines and script commands, but XUnity translates the text line by line when it is shown, so translate those from the runtime log, not from these files. |
| `master_summary.tsv` | strings still missing per table/column | planning. |

Machine translation can be used for a first draft of long descriptions, but a person must check names against the glossary and the
placeholders against the linter.

---

## 3. Translating buttons and other baked text (textures)

Many labels (menu buttons, tabs, banners) are Japanese **drawn into an atlas image**, so `en.txt` cannot translate them. We redraw them:
erase the Japanese glyphs, rebuild the button face underneath, draw the English text in the original colours, and let XUnity load the
edited atlas instead of the original.

### 3.1 Pieces
| Piece | Role |
|---|---|
| `PoK/BepInEx/Translation/en/Texture/<atlas>_png [<hash>].png` | atlas image dumped by XUnity (`EnableTextureDumping`). The edited atlas is written back over this file; XUnity loads a file whose content differs from the original. |
| `translation/atlas/<atlas>.json` | sprite rectangles (name, x, y, width, height) of the atlas. Created by `tools/atlas_dump.py` (atlases in DLC bundles, reads `PoK/Data/paths.json`) or `tools/atlas_dump_raw.py` (atlases inside `PotK_Data/data.unity3d`). |
| `translation/texture_text.json` | the work list: `{ "<atlas>": { "<sprite name>": { "text": "English", ...options } } }`. |
| `translation/texture_orig/` | untouched originals, copied automatically the first time the tool runs. Every run redraws from the original, so you can re-run safely. |
| `tools/texture_translate.py` | applies all specs. `--sheet` writes a before/after sheet to `translation/texture_preview.png`. |
| `tools/tt_one.py <atlas> <out.png> [sprite ...]` | applies one atlas (optionally only some sprites) and writes a **zoomed before/after sheet with centre cross-hairs** (zoom: env `TT_ZOOM`, default 4). Use this while iterating. |
| `tools/make_tab_labels.py` | special case: draws wide standalone labels for sprites too narrow for readable English (Event tabs); the PotkWide plugin lays them over the sprite. |

### 3.2 Steps for a new button
1. **Dump the atlas.** Close the game, then
   ```powershell
   $env:POTK_DUMP_TEXTURES = '1'
   powershell -ExecutionPolicy Bypass -File tools/i18n-sync.ps1
   ```
   Start the game (`play.bat`) and open the screen that has the button. XUnity writes the atlas to `PoK/BepInEx/Translation/en/Texture/`.
   Unset the variable afterwards (dumping slows loading).
2. **Get sprite rectangles** (once per atlas; the file name before `_png` is the atlas name):
   ```powershell
   python -I tools/atlas_dump.py <atlas>          # or: python -I tools/atlas_dump_raw.py <atlas>
   ```
3. **Find the sprites that contain Japanese.** Run `python -I tools/export_button_candidates.py` (or use the ready sheets in
   `extracted/buttons/sheets/`): each sheet shows the not-yet-translated sprites of an atlas with their names. There is no OCR, a person
   decides. `buttons_candidates.csv` lists every sprite with `status` (`done`/`todo`) and a `name_hint` (`yes` when the name looks like a
   label/button). `atlases_without_data.txt` lists GUI atlases that were never extracted.
4. **Write a spec** in `translation/texture_text.json`. The simplest case (a text-only sprite):
   ```json
   "mypage_sozai": { "slc_text_battle": { "text": "BATTLE", "mode": "banner" } }
   ```
   The key is the sprite name without `.png...`. The most useful options:
   | Option | Meaning |
   |---|---|
   | `text` | the English label (required). |
   | `box` | `[x0,y0,x1,y1]` as fractions of the sprite: the area where the Japanese is **erased** (default: the whole sprite). It must cover every glyph but never icons or decoration. |
   | `text_box` | where the English is drawn (defaults to `box`); use it to move the text without changing what is erased. |
   | `mode` | how the Japanese is erased/rebuilt: `banner` (text on a coloured band), `ring` (label over a round icon), `darkbar` (light text on a dark bar), `shelf` (footer labels), `rows` (button whose background is constant along each row), none (clear to transparent). |
   | `fill`, `fill_grad`, `outline`, `shadow`, `outer` | colours; by default they are sampled from the original glyphs. Override only when sampling cannot work. |
   | `center`, `per_line`, `cap_px`, `baseline`, `dy`, `scale`, `pad`, `shear` | placement and size (centre both axes, one style per line, capital height, baseline, vertical nudge, ...). |
   | `inpaint` + `lt`/`dk`/`yellow`, `grow`, `edge`, `kernel`, `diff`, `minc`, `thr`, `radius` | precise glyph mask + inpainting for buttons with gradients. `edge` keeps the rim untouched. |
   Look at neighbouring specs of the same atlas first and copy the closest one; all options are documented at the top of
   `tools/texture_translate.py` and in the `redraw()` function.
5. **Preview before touching the game.**
   ```powershell
   python -I tools/tt_one.py <atlas> preview.png <sprite> [<sprite> ...]
   ```
   Open `preview.png` (original on top, result below, centre cross-hairs). Fix the spec until it passes the checklist in 3.3.
6. **Apply** `python -I tools/texture_translate.py` (all atlases), restart the game, look at the screen.

### 3.3 Quality checklist (the bar the project owner set)
* **Erase first, then rebuild, then write.** Remove the Japanese pixel by pixel (precise glyph + outline/shadow mask, not a big box), repaint
  the button face as if the text had never existed (also redraw any detail the Japanese hid, e.g. a diamond tip), and only then draw the
  English with no background patch under it.
* English uses the **same colours** as the original (fill gradient and outline) and the same outline style as sibling labels.
* **Centred** on the original text footprint / button face (visual centre, e.g. next to an icon), same size class as the Japanese.
  Verify with the cross-hair preview.
* Never cover icons or decoration; keep the rim, shadow and highlight of the button untouched.
* Two-line labels: no all-round outline, use the soft `shadow` option, same style for each line, slightly larger.
* Work in **small groups** (one screen at a time), check the zoomed before/after, then continue.

### 3.4 What is NOT covered
* Images that are not atlas sprites (event banners, full-screen art, gacha/shop banners) need separate handling: dump them with XUnity and
  replace the PNG with the same file name.
* The PotkWide plugin (`tools/PotkWide`) handles layout (16:9), not translation.

---

## 4. Everything still untranslated (snapshot)

**Not covered by these lists:** strings hard-coded in the client code or in scene/prefab labels (they only show up in the runtime log when displayed), text produced by the server, subtitles/voice text, and images that are not atlas sprites. The runtime log (`translation/untranslated.txt`) is the only source for those, so keep playing with the server running and re-run the export.
* Runtime text: `extracted/runtime_untranslated.txt`
* Client UI constants: `extracted/consts_untranslated.tsv`
* Unit/item/gear/ticket names: `extracted/unit_item_names_untranslated.tsv`
* Master-data strings: `extracted/master/*.tsv` and `extracted/master_summary.tsv`
* Buttons/baked text: `extracted/buttons/` (`buttons_candidates.csv`, `sheets/*.png`, `atlases_without_data.txt`)

The snapshot was produced with `tools/export_untranslated.py` and `tools/export_button_candidates.py`; re-run them any time (they read the
current `en.txt` and `texture_text.json`, so finished work disappears from the lists).
The sheets contain crops of the game's artwork: keep them inside the team, do not publish them.

---

## 5. Working with git
* Small commits; run `tools/check.ps1` before pushing (it must report "All checks passed").
* Only touch your own section of `translation/en.txt`; `texture_text.json` entries are per sprite, so conflicts are rare but keep one
  person per atlas.
* Do not edit `translation/texture_orig/` or the dump folder by hand (generated).

## 6. Troubleshooting
| Symptom | Cause / fix |
|---|---|
| New entries do not show up | restart the game, or press ALT+R; run `tools/i18n-sync.ps1` first. A line may be wrong: run the linter. |
| Text still Japanese although listed in `en.txt` | the game text has extra spaces/line breaks: add the exact string from `untranslated.txt`; trailing `\r` and spaces are tolerated by the server, other differences are not. |
| English cut off on a button | shorten it (1-2 words), compare with sibling labels; some labels need `EnableUIResizing` (already on) or a texture redraw. |
| Edited texture not loaded | the atlas must be the exact dump file name in `Translation/en/Texture`; delete the stale dump, re-dump, re-apply. Check `EnableTextureTranslation=True` in `BepInEx/config/AutoTranslatorConfig.ini` (set by `i18n-sync.ps1`). |
| `no dump for <atlas>` when applying | that atlas was never dumped: do step 3.2.1 on the screen that uses it. |
| Git Bash rewrites `/path` arguments | use PowerShell, or prefix with `MSYS_NO_PATHCONV=1`. |
