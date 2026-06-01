---
name: df-compile
description: Compile (build) a DataFlex program with the console compiler (DfCompConsole.exe). Use when the user wants to compile/build a DataFlex .src program, check that DataFlex code compiles, or rebuild an executable. Works for any DataFlex workspace.
---

# Compile a DataFlex program

DataFlex compiles a `.src` program against a workspace (`.sws`) using the console
compiler. Compiling a `.src` transitively compiles every package (`.pkg`) it uses,
so there is no separate build step.

## Compiler

`C:\Program Files\DataFlex <ver>\Bin64\DfCompConsole.exe`

Pick the version that matches the target workspace's `Version=` (see its `.sws` /
`Config.ws`). Installed here: 20.1, 23.0, 24.0, 25.0, 26.0. When unsure, use the
newest version that matches the workspace.

## Command

The two arguments use **different path bases** — this is the easy mistake to make:

- **`-x<workspace>.sws`** — the workspace file. Resolved against your **current working
  directory** (or pass an absolute path). No space after `-x`.
- **`-c <program>.src`** — the program source. Resolved against the **workspace's Home
  directory** (the folder that contains the `.sws`), **not** your current directory.
  Since `AppSrcPath` is normally `.\AppSrc`, this is almost always `AppSrc\<program>.src`
  — with **no** workspace-folder prefix.

```powershell
$compiler = "C:\Program Files\DataFlex 25.0\Bin64\DfCompConsole.exe"
& $compiler -x"<workspace>.sws" -c "AppSrc\<program>.src" 2>&1 | Select-String -Pattern "ERROR:|Total Errors|Total Warnings"
Write-Output "EXIT=$LASTEXITCODE"
```

### Watch out: compiling a workspace that lives in a subfolder

Say the repo root `C:\repo` holds several workspaces and you want to build
`C:\repo\coreM\AppSrc\WebApp.src` in workspace `C:\repo\coreM\coreM.sws`, while staying
at `C:\repo`:

```powershell
& $compiler -x"coreM\coreM.sws" -c "AppSrc\WebApp.src"   # ✅ -c is relative to coreM\
```

- ❌ `-c "coreM\AppSrc\WebApp.src"` is **wrong**: the compiler prepends the workspace
  folder and ends up looking for `coreM\coreM\AppSrc\WebApp.src` → *file not found*.
- `-x` keeps the `coreM\` prefix (it's relative to your current dir); `-c` does not
  (it's relative to the workspace's Home). The mismatch is the whole gotcha.
- The trap only bites when your current dir ≠ the workspace folder. If you `cd` into
  `coreM\` first, both args lose the prefix and `-c "AppSrc\WebApp.src"` still works.

The compiler log is verbose; filtering on `ERROR:|Total Errors|Total Warnings`
keeps it readable. Drop the filter to see the full include trace when diagnosing a
missing-package or ordering problem.

## Reading the result

- **Success** = process `EXIT=0` **and** `Total Errors   : 0`.
- Errors print as `ERROR: <num> ... ON LINE: <n> ... OF FILE: <path>`.
- Warnings are summarised as `Total Warnings : <n>`; pre-existing ones unrelated to
  your change can be ignored.

## Compiling while the app is running (locked binary)

If the program is running (e.g. open in the Studio, or a live web-app process pool),
the linker can't overwrite the `.exe` and you get:

```
LINK ERROR: UNABLE TO OPEN FILE ...\<name>.exe
Total Errors   : 1
```

This is **not** a code error — the source compiled fine (0 source `ERROR: ... ON LINE`
lines); only the final link step failed on the locked file. To get a full clean build
(including link) **without stopping the running app**, compile a throwaway copy of the
`.src` so the linker writes a differently-named `.exe`:

Mind the two path bases here too: `Copy-Item` / `Remove-Item` touch the filesystem so
they take **current-dir-relative** paths (here prefixed with `$wsDir`), while `-c` stays
**workspace-relative** (`AppSrc\...`, no prefix). They only look the same when you run
from inside the workspace folder.

```powershell
$wsDir = "coreM"                  # folder that holds the .sws (current-dir-relative)
$ws    = "$wsDir\coreM.sws"
$compiler = "C:\Program Files\DataFlex 25.0\Bin64\DfCompConsole.exe"
Copy-Item "$wsDir\AppSrc\WebApp.src" "$wsDir\AppSrc\zz_buildcheck.src" -Force
& $compiler -x"$ws" -c "AppSrc\zz_buildcheck.src" 2>&1 | Select-String -Pattern "ERROR:|Total Errors|Total Warnings"
Write-Output "EXIT=$LASTEXITCODE"
# clean up the throwaway artifacts (filesystem paths, so $wsDir-prefixed like the copy)
Remove-Item "$wsDir\AppSrc\zz_buildcheck.src","$wsDir\Programs\zz_buildcheck.exe","$wsDir\Programs\zz_buildcheck.dbg","$wsDir\AppSrc\zz_buildcheck.x64.dep","$wsDir\AppSrc\zz_buildcheck.x64.prn" -Force -ErrorAction SilentlyContinue
```

The output name derives from the `.src` filename, so the copy produces
`zz_buildcheck.exe` (not locked) and links cleanly — giving a true `Total Errors : 0`.
Always delete the `zz_buildcheck.*` files (`.src`, `.exe`, `.dbg`, `.x64.dep`, `.x64.prn`)
afterwards. Alternatively, just close the program in the Studio / stop the process and
compile the real `.src` normally.

## Output

The executable lands in the workspace's `ProgramPath` (commonly `...\Programs\<name>.exe`).
The `64BitSuffix` setting in the program's `.cfg` controls whether the name gets a
`64` suffix.
