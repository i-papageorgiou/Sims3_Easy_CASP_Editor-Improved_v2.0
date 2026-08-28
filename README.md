# Easy CASP Editor — Improved

A tool for editing the Age/Gender, Clothing Type, and Category flags (Everyday, Formalwear,
Valid For Random, etc.) of CAS Parts in **The Sims 3** — hair, clothing, makeup, and accessories —
with multi-facet filtering and bulk editing across any number of items at once.

> **Unofficial, independent rewrite.** This project is based on Kuree's original *Easy CASP
> Editor* (2012) — reverse-engineered from the original compiled `.exe` since no source was
> available — but is **not made, endorsed, or supported by Kuree**. All credit for the original
> concept, UI design, and CASP file-format handling belongs to Kuree. See [Credits](#credits).

## What's new vs. the original

The original let you filter by pack and gender only, and edit one item at a time. This rewrite adds:

- **Multi-facet filtering** — combine Gender, Age, Clothing Type, and Category filters at once
  (e.g. *"Female, Young Adult or Adult, Bottom, Everyday"*), each facet itself multi-select. A
  "Modified only" toggle shows just what you've edited this session; a search box matches by name
  or instance ID.
- **Multi-select + bulk editing** — select any number of items and every flag checkbox becomes
  tri-state (checked / unchecked / mixed). Click one to apply that change to the *whole selection*
  at once.
- **Undo** (Ctrl+Z) for bulk edits.
- **Explicit Save / Save As**, with a save-or-discard prompt on exit and before switching packs —
  the original silently wrote to the Mods folder on almost every action with no way to back out.
- A number of correctness and stability fixes found while rebuilding the original's logic (see
  [Bugs found and fixed](#bugs-found-and-fixed-vs-the-original)).

## Requirements

- Windows with .NET Framework 4.8
- The Sims 3 installed
- The tool is **32-bit** — a requirement of the underlying game-data library (`CASP.dll`), not a
  limitation of your system

## Installation (end users)

Grab the latest zip from [Releases](../../releases), extract the whole folder anywhere, and run
`CaspEditor.exe`. Every file in the folder is required — the DLLs alongside the `.exe` aren't
optional. See the bundled `README.txt` for usage details.

## Building from source

There's no `.sln` — build the project file directly:

```powershell
& "C:\Program Files\Microsoft Visual Studio\2022\<edition>\MSBuild\Current\Bin\MSBuild.exe" `
    src\CaspEditor\CaspEditor.csproj /p:Configuration=Release /p:Platform=x86
```

Requires MSBuild with the .NET Framework 4.8 targeting pack (Visual Studio 2022 Build Tools or
full IDE, any edition). `PlatformTarget` is fixed at `x86` — `CASP.dll` is 32-bit-only, and the
32-bit registry view is also what resolves `HKLM\SOFTWARE\Sims` correctly.

Output goes to `build\CaspEditor.exe` alongside copies of everything under `lib\`.

### Dependencies

`lib/` holds the CASP data-access layer and the [s3pi](http://s3pi.sourceforge.net/) library set —
`CASP.dll`, `s3pi.Interfaces.dll`, `s3pi.Package.dll`, `s3pi.WrapperDealer.dll`,
`s3pi.CASPartResource.dll`, `s3pi.DefaultResource.dll`, `s3pi.Settings.dll`, `System.Custom.dll` —
taken as-is from the original `CASP_Editor/` build and referenced with `HintPath`, not rebuilt.

### Regenerating the decompiled reference (optional)

`tools/decompile.ps1` re-derives readable C# from the original `CASP_Editor.exe` into `ref/` using
[ICSharpCode.Decompiler](https://github.com/icsharpcode/ILSpy), fetched from NuGet on first run.
This is **reference material only** — `ref/` is never part of the build — kept for anyone wanting
to see the original logic this rewrite is based on. One type (`My.Forms.Create__Instance__`, a
VB-compiler-generated designer accessor) fails to decompile; that's expected and harmless.

### Running the validation suite

`tools/validate.ps1` is a non-interactive regression suite that exercises the Model/Editing/Io
layers directly (load, filter, tri-state bulk edit, undo, save round-trip, reset) against your
installed game data — see [Testing](#testing) below.

## Project structure

```
CASP_Editor/            Original compiled Easy CASP Editor (untouched, still runnable)
CASP_Editor_backup/      Backup copy of the original (untouched)
lib/                     CASP.dll + s3pi.*.dll, referenced by the build, not rebuilt
ref/                     Decompiled original source -- reference only, never built
tools/
  decompile.ps1          Regenerates ref/ from CASP_Editor.exe
  validate.ps1           Non-interactive regression suite (see Testing)
src/CaspEditor/
  CaspEditor.csproj
  Program.cs
  Model/                 CaspItem, CaspLibrary, CaspFilter, PackCatalog
  Editing/               FlagOps, BulkEditor, UndoStack
  Io/                    OverridePackageWriter
  Ui/                    MainForm, FilterPanel, FlagEditorPanel, ...
build/                   Build output (CaspEditor.exe + copied DLLs)
release/                 Packaged zip(s) for distribution
Project_Plan.md          Design/implementation notes from the original rebuild
```

## Architecture

- **Model** — `CaspItem` wraps one CAS part (the live `CASPartResource`, its source resource
  key, thumbnails, and a baseline snapshot of the four editable fields for cheap modified/revert
  checks). `CaspLibrary` loads a pack once into memory; `CaspFilter` matches items against the
  active Gender/Age/Type/Category/Search facets (OR within a facet, AND across facets).
- **Editing** — `FlagOps` does the actual bitwise mutation per flag kind. `BulkEditor` derives
  tri-state display and applies a mutation across a whole selection. `UndoStack` snapshots and
  restores a batch.
- **Io** — `OverridePackageWriter` writes changed items into the Mods-folder override package
  (matching the original's approach: `AddResource` keyed by the *original* source resource entry,
  so the game treats it as an override).
- **Ui** — plain hand-authored WinForms, no designer files.

## Testing

`tools/validate.ps1` runs entirely outside the UI (nothing in this environment could drive the
WinForms UI itself), exercising the real classes against your actually-installed game data:

1. Load a real pack via `CaspLibrary`
2. `IsModified` performance regression guard
3. `FlagOps` set/clear correctness
4. `RebaseOriginal` / `Revert`
5. `BulkEditor` tri-state + `UndoStack`
6. Save round-trip to a **scratch** package (never your real Mods folder), independently re-read
   with a fresh `s3pi` package open to confirm the override identity and flags match exactly
7. `ResetAll`

Must run under 32-bit PowerShell (it re-launches itself under SysWOW64 automatically if started
from a 64-bit host):

```powershell
tools\validate.ps1
```

## Bugs found and fixed (vs. the original)

Found while reverse-engineering the original's IL to rebuild its load/save logic:

- Age/Gender edits used arithmetic `+`/`-` narrowed through a checked byte conversion — clearing a
  bit that wasn't set could corrupt neighboring bits or throw `OverflowException`. Clothing Type
  used arithmetic subtraction on a non-flag scalar enum. Fixed to real bitwise Or/And-Not (or plain
  assignment for the scalar Clothing Type).
- The original's checkbox-driven filter matched checkbox *captions* as substrings against a
  resource's `ToString()` dump — case-sensitive, so `"Adult"` matched inside `"YoungAdult"`, and it
  broke entirely under a non-English language pack. Replaced with real bitwise/equality tests.
- A locale-lookup crash (`KeyNotFoundException` in `CASP.dll`'s `ModFolder`) on systems whose
  registered Sims 3 locale string doesn't hyphenate the way the original's hardcoded table expects
  (e.g. `en_US` vs. `en-US`) — reproduced against the original `CASP_Editor.exe` itself, not
  something this rewrite introduced. Worked around with a documented fallback path.
- Eagerly decoding every thumbnail resource (a pack can have several thumbnail entries per CAS
  part) into full-resolution bitmaps and holding them all in memory could `OutOfMemoryException` a
  large pack in this 32-bit process. Fixed to decode one icon-sized thumbnail per item eagerly and
  the rest lazily on demand.
- `IsModified` originally hashed a full `ToString()` dump of the resource (a reflection-driven text
  dump including nested lists) to detect changes — measured at ~7.4ms/item, ~11.5s across a
  1561-item pack, called on every selection change and filter change. Replaced with direct
  comparison of the four fields this editor can actually mutate — same correctness, ~100x cheaper.

## Known limitations

- Only Age, Gender, Clothing Type, and Category flags are editable — the same scope as the
  original tool. Mesh, texture, and other CASP data are untouched.
- Tested against a full base-game library (~1500 items); a very large multi-pack custom-content
  library will take longer to load proportionally.

## Credits

- Original *Easy CASP Editor* concept, UI, and CASP file-format handling: **Kuree** (2012)
- [s3pi](http://s3pi.sourceforge.net/) library set for Sims 3 package/resource handling
