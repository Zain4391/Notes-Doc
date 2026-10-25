# PowerShell Commands Reference

Documentation for the PowerShell commands (cmdlets) used most often, with every common parameter ("flag") explained. For quick copy-paste see the examples under each command.

**Contents**

[Basics](#basics) · [Getting Help](#getting-help) · [Navigation](#navigation) · [Files & Folders](#files--folders) · [Reading & Writing Content](#reading--writing-content) · [Searching & Filtering](#searching--filtering) · [Pipeline & Output](#pipeline--output) · [Processes & Services](#processes--services) · [System & Network](#system--network) · [Environment & Variables](#environment--variables) · [Execution Policy & Scripts](#execution-policy--scripts) · [Packages & Modules](#packages--modules) · [Web & Remoting](#web--remoting) · [Common Parameters](#common-parameters) · [Aliases](#aliases)

---

## Basics

### Command anatomy

PowerShell commands are called **cmdlets** and follow a `Verb-Noun` pattern.

```powershell
Get-ChildItem -Path C:\Projects -Recurse -Filter *.md
#  │            │     │          │        └─ Parameter with a value
#  │            │     │          └─ Switch parameter (no value, on/off)
#  │            │     └─ Parameter with a value
#  │            └─ Parameter name (starts with a dash)
#  └─ Cmdlet (Verb-Noun)
```

| Term | Meaning |
|------|---------|
| **Cmdlet** | A built-in PowerShell command, e.g. `Get-Process`. |
| **Parameter** | An option that takes a value, e.g. `-Path C:\temp`. |
| **Switch** | A parameter with no value; its presence turns it on, e.g. `-Recurse`. |
| **Alias** | A short name for a cmdlet, e.g. `ls` for `Get-ChildItem`. |
| **Pipeline (`\|`)** | Sends the output *objects* of one command into the next. |

> PowerShell passes **objects**, not text, through the pipeline. That is the main difference from bash.

> Parameter names are **case-insensitive** and can be shortened as long as they stay unambiguous (`-Rec` works for `-Recurse`).

---

## Getting Help

### `Get-Help`

Shows documentation for a cmdlet.

```powershell
Get-Help Get-ChildItem
Get-Help Get-ChildItem -Examples
Get-Help Get-ChildItem -Full
Get-Help about_*
```

| Flag | Description |
|------|-------------|
| `-Name` | Command to look up (positional, so the name can be given without `-Name`). |
| `-Examples` | Show only usage examples. |
| `-Detailed` | Parameter descriptions plus examples. |
| `-Full` | Everything: syntax, parameters, inputs/outputs, notes. |
| `-Parameter <name>` | Show help for one parameter only. `*` lists all. |
| `-Online` | Open the latest docs on Microsoft Learn in the browser. |
| `-ShowWindow` | Open help in a searchable window (Windows PowerShell). |

Update local help files: `Update-Help` (run as administrator).

### `Get-Command`

Finds commands by name, verb, noun or module.

```powershell
Get-Command *process*
Get-Command -Verb Get -Noun Process
Get-Command -Module Microsoft.PowerShell.Management
```

| Flag | Description |
|------|-------------|
| `-Name` | Name or wildcard pattern to match. |
| `-Verb` | Filter by verb (`Get`, `Set`, `New`, `Remove`…). |
| `-Noun` | Filter by noun (`Process`, `Item`, `Service`…). |
| `-Module` | Only commands from the given module. |
| `-CommandType` | `Cmdlet`, `Function`, `Alias`, `Application`, … |

### `Get-Member`

Lists the properties and methods of whatever object is piped into it. Essential for discovering what you can work with.

```powershell
Get-Process | Get-Member
Get-Process | Get-Member -MemberType Property
```

| Flag | Description |
|------|-------------|
| `-MemberType` | Limit output to `Property`, `Method`, `NoteProperty`, etc. |
| `-Name` | Show only members matching this name/wildcard. |
| `-Static` | Show static members of the type instead of instance members. |

---

## Navigation

### `Get-Location` (alias `pwd`)

Prints the current directory.

```powershell
Get-Location
```

### `Set-Location` (aliases `cd`, `chdir`)

Changes the current directory.

```powershell
Set-Location D:\Docs
cd ..          # Up one level
cd ~           # Home directory
cd -           # Previous directory (PowerShell 6.2+)
```

| Flag | Description |
|------|-------------|
| `-Path` | Target directory (supports wildcards). |
| `-LiteralPath` | Target directory with **no** wildcard interpretation. Use when the name contains `[` or `]`. |
| `-PassThru` | Also output the new location object. |

### `Push-Location` / `Pop-Location`

Saves the current directory on a stack (`pushd`) and returns to it later (`popd`).

```powershell
Push-Location C:\Windows
Pop-Location
```

---

## Files & Folders

### `Get-ChildItem` (aliases `ls`, `dir`, `gci`)

Lists files and folders.

```powershell
Get-ChildItem
Get-ChildItem -Path D:\Docs -Recurse -Filter *.md
Get-ChildItem -Force -Hidden
Get-ChildItem -Directory -Depth 1
```

| Flag | Description |
|------|-------------|
| `-Path` | Location to list (default: current directory). Wildcards allowed. |
| `-LiteralPath` | Same as `-Path`, but wildcards are treated as normal characters. |
| `-Filter` | Filter applied by the provider; fast. Single pattern like `*.txt`. |
| `-Include` | Pattern(s) to include. Works with `-Recurse` or a wildcard path. Slower than `-Filter`. |
| `-Exclude` | Pattern(s) to leave out. |
| `-Recurse` | Include all subfolders. |
| `-Depth <n>` | With `-Recurse`, limit how many levels deep to go. |
| `-Force` | Include hidden and system items. |
| `-Hidden` | Show *only* hidden items. |
| `-File` | Files only. |
| `-Directory` | Folders only. |
| `-Name` | Return only names (strings) instead of objects. |

### `New-Item` (alias `ni`)

Creates a file, folder, or other item.

```powershell
New-Item -Path .\Powershell -ItemType Directory
New-Item -Path .\notes.md -ItemType File -Value "# Notes"
New-Item -Path .\a\b\c -ItemType Directory -Force
```

| Flag | Description |
|------|-------------|
| `-Path` | Where to create the item. |
| `-Name` | Name of the item (combine with `-Path` as the parent). |
| `-ItemType` | `File`, `Directory`, `SymbolicLink`, `Junction`, `HardLink`. |
| `-Value` | Initial content (for files) or link target. |
| `-Force` | Overwrite an existing file; create missing parent folders. |

`mkdir` is a function that wraps `New-Item -ItemType Directory`.

### `Copy-Item` (aliases `cp`, `copy`, `cpi`)

```powershell
Copy-Item .\file.txt .\backup\file.txt
Copy-Item .\src -Destination .\dest -Recurse
```

| Flag | Description |
|------|-------------|
| `-Path` | Source item(s). |
| `-Destination` | Where to copy to. |
| `-Recurse` | Copy folder contents including subfolders. Without it, only the empty folder is copied. |
| `-Force` | Overwrite read-only or existing files. |
| `-Container` | Preserve folder structure when copying (default `$true`). |
| `-PassThru` | Output the copied item(s). |

### `Move-Item` (aliases `mv`, `move`, `mi`)

Moves **or renames** an item.

```powershell
Move-Item .\old.txt .\archive\old.txt
Move-Item .\old.txt .\new.txt     # Rename
```

| Flag | Description |
|------|-------------|
| `-Path` | Item to move. |
| `-Destination` | New location or name. |
| `-Force` | Overwrite an existing destination. |
| `-PassThru` | Output the moved item. |

### `Rename-Item` (aliases `ren`, `rni`)

```powershell
Rename-Item -Path .\old.md -NewName new.md
```

| Flag | Description |
|------|-------------|
| `-Path` | Item to rename. |
| `-NewName` | New name only (not a full path). |
| `-Force` | Allow renaming of items that would otherwise be blocked. |

### `Remove-Item` (aliases `rm`, `del`, `rd`, `ri`)

> ⚠️ Deletion is permanent: nothing goes to the Recycle Bin.

```powershell
Remove-Item .\temp.txt
Remove-Item .\build -Recurse -Force
Remove-Item .\*.log -WhatIf
```

| Flag | Description |
|------|-------------|
| `-Path` | Item(s) to delete. Wildcards allowed. |
| `-Recurse` | Delete folder contents without a confirmation prompt per child. |
| `-Force` | Delete hidden, read-only items. |
| `-Include` / `-Exclude` | Pattern filters (see `Get-ChildItem`). |
| `-WhatIf` | **Dry run**: show what would be deleted without doing it. |
| `-Confirm` | Prompt before each deletion. |

### `Test-Path`

Returns `$true` / `$false` depending on whether a path exists.

```powershell
Test-Path D:\Docs\Powershell
Test-Path .\file.txt -PathType Leaf
```

| Flag | Description |
|------|-------------|
| `-PathType` | `Container` (folder), `Leaf` (file), or `Any`. |
| `-IsValid` | Only check that the path *syntax* is valid. |

### `Get-Item` / `Get-ItemProperty` / `Set-ItemProperty`

Get a single item, or read/change its properties (attributes, registry values).

```powershell
Get-Item .\file.txt | Select-Object Name, Length, LastWriteTime
Set-ItemProperty -Path .\file.txt -Name IsReadOnly -Value $true
```

---

## Reading & Writing Content

### `Get-Content` (aliases `cat`, `type`, `gc`)

```powershell
Get-Content .\log.txt
Get-Content .\log.txt -Tail 20 -Wait
Get-Content .\big.txt -TotalCount 5
```

| Flag | Description |
|------|-------------|
| `-Path` | File(s) to read. |
| `-TotalCount <n>` (alias `-Head`) | Read only the first *n* lines. |
| `-Tail <n>` | Read only the last *n* lines. |
| `-Wait` | Keep the file open and print new lines as they are written (like `tail -f`). |
| `-Raw` | Return the whole file as one string instead of an array of lines. |
| `-Encoding` | `utf8`, `utf8BOM`, `ascii`, `unicode`, … |
| `-ReadCount <n>` | Lines sent down the pipeline at once. `0` = all at once. |

### `Set-Content` (alias `sc`)

Writes content, **replacing** whatever was in the file.

```powershell
Set-Content -Path .\notes.md -Value "# Hello"
```

### `Add-Content` (alias `ac`)

Appends content to the end of a file.

```powershell
Add-Content -Path .\notes.md -Value "`nMore text"
```

| Flag (both) | Description |
|------|-------------|
| `-Path` | Target file. |
| `-Value` | Content to write. |
| `-Encoding` | Output encoding. |
| `-NoNewline` | Don't add a trailing newline. |
| `-Force` | Write to read-only files. |

### `Out-File` and redirection

```powershell
Get-Process | Out-File .\procs.txt
Get-Process | Out-File .\procs.txt -Append
"text" > file.txt     # Overwrite (same as Out-File)
"text" >> file.txt    # Append
```

| Flag | Description |
|------|-------------|
| `-FilePath` | Destination file. |
| `-Append` | Add to the end instead of overwriting. |
| `-Encoding` | Output encoding. |
| `-Width <n>` | Characters per line; prevents truncated table output. |
| `-NoClobber` | Refuse to overwrite an existing file. |

> In Windows PowerShell 5.1, `>` and `Out-File` write **UTF-16** by default, which surprises many tools. PowerShell 7 defaults to UTF-8 (no BOM).

---

## Searching & Filtering

### `Select-String` (alias `sls`)

PowerShell's `grep`. Searches text using regular expressions.

```powershell
Select-String -Path .\*.md -Pattern "docker"
Get-ChildItem -Recurse -Filter *.cs | Select-String "TODO" -CaseSensitive
```

| Flag | Description |
|------|-------------|
| `-Pattern` | Regex (or text) to look for. |
| `-Path` | File(s) to search. |
| `-SimpleMatch` | Treat the pattern as plain text, not a regex. |
| `-CaseSensitive` | Match case (default is case-insensitive). |
| `-NotMatch` | Return lines that do **not** match. |
| `-Context <before>,<after>` | Include surrounding lines. |
| `-List` | Only first match per file. |
| `-Quiet` | Return `$true`/`$false` instead of match objects. |
| `-Include` / `-Exclude` | Filter files by pattern. |

### `Where-Object` (alias `where`, `?`)

Filters objects in the pipeline.

```powershell
Get-Process | Where-Object CPU -gt 100
Get-ChildItem | Where-Object { $_.Length -gt 1MB -and $_.Extension -eq ".log" }
```

| Comparison operator | Meaning |
|------|---------|
| `-eq` / `-ne` | Equal / not equal |
| `-gt` / `-ge` | Greater than / greater or equal |
| `-lt` / `-le` | Less than / less or equal |
| `-like` / `-notlike` | Wildcard match (`*`, `?`) |
| `-match` / `-notmatch` | Regex match |
| `-contains` / `-in` | Collection membership |
| `-and`, `-or`, `-not` | Logical operators |

`$_` (or `$PSItem`) is the current object in the pipeline.

### `Select-Object` (alias `select`)

Chooses properties or limits the number of objects.

```powershell
Get-Process | Select-Object Name, Id, CPU -First 10
Get-ChildItem | Select-Object -ExpandProperty Name
```

| Flag | Description |
|------|-------------|
| `-Property` | Properties to keep. Accepts calculated properties. |
| `-ExpandProperty` | Return the raw *value* of a property rather than an object wrapping it. |
| `-First <n>` / `-Last <n>` | Keep first/last *n* objects. |
| `-Skip <n>` | Skip the first *n* objects. |
| `-Unique` | Remove duplicates. |

### `Sort-Object` (alias `sort`)

```powershell
Get-Process | Sort-Object CPU -Descending
```

| Flag | Description |
|------|-------------|
| `-Property` | Property(ies) to sort by. |
| `-Descending` | Reverse order (default is ascending). |
| `-Unique` | Drop duplicate values. |
| `-Top <n>` / `-Bottom <n>` | Return only the top/bottom *n* (PowerShell 6.2+). |

### `Group-Object`, `Measure-Object`, `ForEach-Object`

```powershell
Get-ChildItem -Recurse | Group-Object Extension
Get-Content .\file.txt | Measure-Object -Line -Word -Character
Get-ChildItem *.txt | ForEach-Object { $_.Name.ToUpper() }
```

| Cmdlet | Key flags |
|--------|-----------|
| `Group-Object` | `-Property` group key; `-NoElement` omit the grouped items, show counts only. |
| `Measure-Object` | `-Sum`, `-Average`, `-Minimum`, `-Maximum`, `-Line`, `-Word`, `-Character`, `-Property`. |
| `ForEach-Object` (`%`) | `-Process` script block per item; `-Begin` / `-End` run once before/after; `-Parallel` run iterations concurrently (PowerShell 7+); `-ThrottleLimit` max concurrent runs. |

---

## Pipeline & Output

| Cmdlet | Purpose | Notable flags |
|--------|---------|---------------|
| `Format-Table` (`ft`) | Show as a table | `-AutoSize` fit columns to content, `-Wrap` wrap long text, `-Property` choose columns |
| `Format-List` (`fl`) | Show every property as a list | `-Property *` show all properties |
| `Out-String` | Convert objects to a string | `-Stream` one string per line |
| `Out-Host -Paging` | Page long output | `-Paging` |
| `Out-GridView` (`ogv`) | Interactive filterable window (Windows) | `-PassThru` return the rows you select |
| `Out-Null` | Discard output | — |
| `Tee-Object` (`tee`) | Display *and* save output | `-FilePath`, `-Append`, `-Variable` |
| `Write-Host` | Print directly to the console | `-ForegroundColor`, `-BackgroundColor`, `-NoNewline` |
| `Write-Output` | Send to the pipeline | `-NoEnumerate` |
| `Write-Error` / `Write-Warning` / `Write-Verbose` | Streams for errors, warnings and verbose info | — |
| `ConvertTo-Json` | Convert objects to JSON | `-Depth <n>` (default is only 2!), `-Compress` |
| `ConvertFrom-Json` | Parse JSON into objects | `-AsHashtable` |
| `Export-Csv` / `Import-Csv` | Write / read CSV | `-NoTypeInformation` (5.1), `-Delimiter`, `-Encoding`, `-Append` |

```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 | Format-Table Name, CPU -AutoSize
Get-Service | Export-Csv .\services.csv -NoTypeInformation
$config = Get-Content .\config.json -Raw | ConvertFrom-Json
```

---

## Processes & Services

### `Get-Process` (alias `ps`, `gps`)

```powershell
Get-Process
Get-Process -Name chrome
Get-Process -Id 1234 -IncludeUserName
```

| Flag | Description |
|------|-------------|
| `-Name` | Process name(s); wildcards OK. |
| `-Id` | Process ID(s). |
| `-IncludeUserName` | Add the owner (requires elevation). |
| `-Module` | List loaded modules. |
| `-FileVersionInfo` | Show file version information. |

### `Start-Process` (alias `saps`, `start`)

Launches a program.

```powershell
Start-Process notepad
Start-Process powershell -Verb RunAs
Start-Process .\setup.exe -ArgumentList "/silent" -Wait
```

| Flag | Description |
|------|-------------|
| `-FilePath` | Executable or document to open. |
| `-ArgumentList` | Arguments passed to the program. |
| `-Wait` | Block until the process exits. |
| `-Verb RunAs` | Run elevated (as administrator). |
| `-WorkingDirectory` | Starting directory. |
| `-NoNewWindow` | Run inside the current console. |
| `-WindowStyle` | `Normal`, `Hidden`, `Minimized`, `Maximized`. |
| `-PassThru` | Return the process object. |
| `-RedirectStandardOutput` / `-RedirectStandardError` | Send streams to files. |

### `Stop-Process` (alias `kill`, `spps`)

```powershell
Stop-Process -Name notepad
Stop-Process -Id 1234 -Force
```

| Flag | Description |
|------|-------------|
| `-Id` / `-Name` | Which process to stop. |
| `-Force` | Terminate without asking. |
| `-PassThru` | Return the stopped process object. |
| `-WhatIf` / `-Confirm` | Preview / confirm. |

### Services

```powershell
Get-Service
Get-Service -Name "wuauserv"
Start-Service -Name Spooler
Stop-Service  -Name Spooler -Force
Restart-Service -Name Spooler
Set-Service   -Name Spooler -StartupType Disabled
```

| Flag | Description |
|------|-------------|
| `-Name` / `-DisplayName` | Select a service by short or display name. |
| `-Force` | Also stop dependent services. |
| `-StartupType` | `Automatic`, `Manual`, `Disabled` (`Set-Service`). |
| `-Status` | Desired state (`Set-Service`). |

---

## System & Network

| Command | Purpose | Notable flags |
|---------|---------|---------------|
| `Get-ComputerInfo` | OS, hardware, BIOS details | `-Property` |
| `$PSVersionTable` | PowerShell version | — |
| `Get-Date` | Current date/time | `-Format "yyyy-MM-dd"`, `-UFormat` |
| `Get-HotFix` | Installed updates | — |
| `Get-Volume` / `Get-PSDrive` | Disks and drives | — |
| `Test-Connection` (`ping`) | ICMP ping | `-Count`, `-Quiet`, `-TargetName` (7+) or `-ComputerName` (5.1) |
| `Test-NetConnection` | Ping / TCP port test / traceroute (Windows) | `-ComputerName`, `-Port`, `-TraceRoute`, `-InformationLevel Detailed` |
| `Resolve-DnsName` | DNS lookup | `-Name`, `-Type` (A, AAAA, MX, TXT…), `-Server` |
| `Get-NetIPAddress` | IP configuration | `-AddressFamily IPv4` |
| `Get-NetTCPConnection` | Open TCP connections (like `netstat`) | `-State Listen`, `-LocalPort`, `-OwningProcess` |

```powershell
Test-NetConnection google.com -Port 443
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
Get-Date -Format "yyyy-MM-dd HH:mm"
```

---

## Environment & Variables

```powershell
$name = "value"                 # Create a variable
$env:PATH                       # Read an environment variable
$env:MY_VAR = "hello"           # Set (current session only)
Get-ChildItem Env:              # List all environment variables
Get-Variable                    # List PowerShell variables
Remove-Variable name
[Environment]::SetEnvironmentVariable("MY_VAR", "hello", "User")   # Persist for the user
```

| Syntax | Meaning |
|--------|---------|
| `$var` | Variable |
| `$env:NAME` | Environment variable |
| `$_` / `$PSItem` | Current pipeline object |
| `$?` | `$true` if the last command succeeded |
| `$LASTEXITCODE` | Exit code of the last native program |
| `$PROFILE` | Path to your PowerShell profile script |
| `$HOME` | User home directory |
| `` ` `` (backtick) | Escape character, and line continuation |
| `"$var"` vs `'$var'` | Double quotes expand variables; single quotes are literal |

---

## Execution Policy & Scripts

### `Get-ExecutionPolicy` / `Set-ExecutionPolicy`

Controls whether scripts (`.ps1`) may run on Windows.

```powershell
Get-ExecutionPolicy -List
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

| Flag | Description |
|------|-------------|
| `-ExecutionPolicy` | `Restricted` (no scripts), `AllSigned`, `RemoteSigned`, `Unrestricted`, `Bypass`. |
| `-Scope` | `Process`, `CurrentUser`, `LocalMachine`. Prefer `CurrentUser` or `Process`. |
| `-Force` | Skip the confirmation prompt. |
| `-List` (`Get`) | Show policy for every scope. |

`RemoteSigned` is the usual developer setting: local scripts run; downloaded ones must be signed.

### Running scripts

```powershell
.\script.ps1                     # Run a script in the current directory (the .\ is required)
. .\script.ps1                   # Dot-source: run in the current scope so functions/variables remain
& "C:\Program Files\App\app.exe" # Call operator: run a command whose path is in a string
```

### Launching PowerShell itself

```powershell
pwsh -NoProfile -File .\script.ps1 -Name "Zain"
powershell -ExecutionPolicy Bypass -Command "Get-Date"
```

| Flag | Description |
|------|-------------|
| `-File` | Run a script file. |
| `-Command` / `-c` | Run a command string. |
| `-NoProfile` | Skip loading the profile (faster, predictable). |
| `-NoExit` | Stay open after running the command. |
| `-ExecutionPolicy` | Override the policy for this session. |
| `-NonInteractive` | Never prompt the user. |
| `-WindowStyle Hidden` | Hide the window. |

`pwsh` is PowerShell 7+; `powershell` is Windows PowerShell 5.1.

---

## Packages & Modules

### Modules

```powershell
Get-Module -ListAvailable
Install-Module -Name PSReadLine -Scope CurrentUser -Force
Import-Module posh-git
Update-Module PSReadLine
Uninstall-Module PSReadLine
```

| Flag | Description |
|------|-------------|
| `-ListAvailable` | List installed modules, not just the loaded ones. |
| `-Scope CurrentUser` | Install without administrator rights. |
| `-Force` | Reinstall / skip prompts. |
| `-AllowClobber` | Install even if a command name already exists. |
| `-RequiredVersion` / `-MinimumVersion` | Pin to a version. |
| `-Repository` | Source repository (default `PSGallery`). |

### WinGet (Windows Package Manager)

```powershell
winget search git
winget install --id Git.Git -e
winget upgrade --all
winget list
winget uninstall --id Git.Git
```

| Flag | Description |
|------|-------------|
| `--id` | Install by exact package ID. |
| `-e`, `--exact` | Exact match on the query. |
| `-s`, `--source` | Choose source (`winget`, `msstore`). |
| `-v`, `--version` | Specific version. |
| `--silent` | No installer UI. |
| `--accept-package-agreements` / `--accept-source-agreements` | Skip license prompts (useful in scripts). |
| `--all` | Apply to all packages (`upgrade`). |

---

## Web & Remoting

### `Invoke-WebRequest` (alias `iwr`, `curl` on 5.1) and `Invoke-RestMethod` (alias `irm`)

`Invoke-WebRequest` returns the full response; `Invoke-RestMethod` parses JSON/XML into objects automatically.

```powershell
Invoke-WebRequest -Uri https://example.com/file.zip -OutFile file.zip
Invoke-RestMethod -Uri https://api.example.com/items -Method Post `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body (@{ name = "test" } | ConvertTo-Json)
```

| Flag | Description |
|------|-------------|
| `-Uri` | URL to call. |
| `-Method` | `Get`, `Post`, `Put`, `Delete`, `Patch`, … |
| `-Headers` | Hashtable of HTTP headers. |
| `-Body` | Request payload. |
| `-ContentType` | Content-Type header, e.g. `application/json`. |
| `-OutFile` | Save the response to a file. |
| `-Credential` | PSCredential for authentication. |
| `-UseBasicParsing` | Skip the IE engine (needed on old 5.1 setups). |
| `-TimeoutSec` | Give up after n seconds. |
| `-SkipCertificateCheck` | Ignore TLS errors (7+, use only for local testing). |

### Remoting

```powershell
Enter-PSSession -ComputerName server01
Invoke-Command -ComputerName server01 -ScriptBlock { Get-Service }
```

| Flag | Description |
|------|-------------|
| `-ComputerName` | Remote machine. |
| `-Credential` | Account to use. |
| `-ScriptBlock` | Code to run remotely. |
| `-ArgumentList` | Values passed into the script block. |
| `-HostName` / `-UserName` | SSH-based remoting (PowerShell 7). |

---

## Common Parameters

Available on almost every cmdlet.

| Parameter | Description |
|-----------|-------------|
| `-WhatIf` | Dry run: report what *would* happen, change nothing. |
| `-Confirm` | Ask for confirmation before acting. |
| `-Verbose` | Show extra progress/diagnostic messages. |
| `-Debug` | Show debug messages (may pause for input). |
| `-ErrorAction` | What to do on a non-terminating error: `Continue` (default), `Stop`, `SilentlyContinue`, `Ignore`, `Inquire`. |
| `-ErrorVariable` (`-EV`) | Store errors in a variable. |
| `-WarningAction` | Same idea for warnings. |
| `-InformationAction` | Control information-stream output. |
| `-OutVariable` (`-OV`) | Copy output into a variable *and* the pipeline. |
| `-OutBuffer` | Batch size before sending to the next command. |
| `-PipelineVariable` (`-PV`) | Name the current object so later pipeline stages can use it. |

```powershell
Remove-Item .\old -Recurse -WhatIf
Get-Item .\missing.txt -ErrorAction SilentlyContinue
try { Get-Item .\missing.txt -ErrorAction Stop } catch { Write-Host "Not found" }
```

### Preference variables

| Variable | Effect |
|----------|--------|
| `$ErrorActionPreference` | Default for `-ErrorAction` in the whole session (`Stop` is handy in scripts). |
| `$VerbosePreference` | Set to `Continue` to show verbose output everywhere. |
| `$ProgressPreference` | Set to `SilentlyContinue` to speed up `Invoke-WebRequest` downloads in 5.1. |

---

## Aliases

| Alias | Cmdlet |
|-------|--------|
| `ls`, `dir`, `gci` | `Get-ChildItem` |
| `cd`, `chdir`, `sl` | `Set-Location` |
| `pwd`, `gl` | `Get-Location` |
| `cat`, `type`, `gc` | `Get-Content` |
| `cp`, `copy`, `cpi` | `Copy-Item` |
| `mv`, `move`, `mi` | `Move-Item` |
| `rm`, `del`, `ri` | `Remove-Item` |
| `ren`, `rni` | `Rename-Item` |
| `echo` | `Write-Output` |
| `cls`, `clear` | `Clear-Host` |
| `ps`, `gps` | `Get-Process` |
| `kill` | `Stop-Process` |
| `sls` | `Select-String` |
| `?` | `Where-Object` |
| `%` | `ForEach-Object` |
| `select` | `Select-Object` |
| `sort` | `Sort-Object` |
| `ft`, `fl` | `Format-Table`, `Format-List` |
| `iwr`, `irm` | `Invoke-WebRequest`, `Invoke-RestMethod` |
| `h`, `history` | `Get-History` |

Look up any alias with `Get-Alias <name>` or `Get-Alias -Definition Get-ChildItem`.

> In **scripts**, prefer full cmdlet names over aliases for readability. Aliases are fine interactively.

---

## Handy One-Liners

```powershell
# Largest 10 files under the current folder
Get-ChildItem -Recurse -File | Sort-Object Length -Descending | Select-Object -First 10 Name, @{n="MB";e={[math]::Round($_.Length/1MB,2)}}

# Find which process is using a port
Get-NetTCPConnection -LocalPort 3000 | Select-Object OwningProcess | ForEach-Object { Get-Process -Id $_.OwningProcess }

# Delete all node_modules folders
Get-ChildItem -Recurse -Directory -Filter node_modules | Remove-Item -Recurse -Force -WhatIf

# Count lines of code in all .cs files
(Get-ChildItem -Recurse -Filter *.cs | Get-Content | Measure-Object -Line).Lines

# Add a folder to the user PATH permanently
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\tools", "User")

# Show command history and re-run entry 5
Get-History; Invoke-History 5
```

---

## See Also

- Official docs: <https://learn.microsoft.com/powershell/>
- In-shell: `Get-Help <cmdlet> -Examples`, `Get-Command`, `Get-Member`
