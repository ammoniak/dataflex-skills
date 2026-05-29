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

Run from the repo / workspace root. `-x` = workspace (`.sws`), `-c` = source to
compile (path relative to the workspace's AppSrc).

```powershell
$compiler = "C:\Program Files\DataFlex 25.0\Bin64\DfCompConsole.exe"
& $compiler -x"<workspace>.sws" -c "AppSrc\<program>.src" 2>&1 | Select-String -Pattern "ERROR:|Total Errors|Total Warnings"
Write-Output "EXIT=$LASTEXITCODE"
```

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

```powershell
$ws   = "<workspace>.sws"
$src  = "AppSrc\<program>.src"
$tmp  = "AppSrc\zz_buildcheck.src"
Copy-Item $src $tmp -Force
$compiler = "C:\Program Files\DataFlex 25.0\Bin64\DfCompConsole.exe"
& $compiler -x"$ws" -c "$tmp" 2>&1 | Select-String -Pattern "ERROR:|Total Errors|Total Warnings"
Write-Output "EXIT=$LASTEXITCODE"
# clean up the throwaway artifacts (src + build outputs)
Remove-Item $tmp -Force -ErrorAction SilentlyContinue
Remove-Item "Programs\zz_buildcheck.exe","Programs\zz_buildcheck.dbg","AppSrc\zz_buildcheck.x64.dep","AppSrc\zz_buildcheck.x64.prn" -Force -ErrorAction SilentlyContinue
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
