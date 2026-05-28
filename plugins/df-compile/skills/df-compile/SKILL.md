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

## Output

The executable lands in the workspace's `ProgramPath` (commonly `...\Programs\<name>.exe`).
The `64BitSuffix` setting in the program's `.cfg` controls whether the name gets a
`64` suffix.
