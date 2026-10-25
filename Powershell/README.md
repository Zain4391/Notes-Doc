# PowerShell

Reference notes for the PowerShell commands used day to day. Each command is documented with a short description, examples, and a table explaining every commonly used flag (parameter).

Commands are written for **PowerShell 7+ (`pwsh`)** and **Windows PowerShell 5.1 (`powershell`)**. Where behaviour differs between the two (for example default file encoding), it is noted in the text.

---

## Topics

| Note | Covers |
|---|---|
| [PowerShell Commands](powershell-commands.md) | Full command reference with flag explanations, grouped by category |

## What's inside the command reference

| Section | Covers |
|---|---|
| [Basics](powershell-commands.md#basics) | Cmdlet anatomy (`Verb-Noun`), parameters vs switches, the object pipeline |
| [Getting Help](powershell-commands.md#getting-help) | `Get-Help`, `Get-Command`, `Get-Member` |
| [Navigation](powershell-commands.md#navigation) | `Get-Location`, `Set-Location`, `Push-Location` / `Pop-Location` |
| [Files & Folders](powershell-commands.md#files--folders) | `Get-ChildItem`, `New-Item`, `Copy-Item`, `Move-Item`, `Remove-Item`, `Test-Path` |
| [Reading & Writing Content](powershell-commands.md#reading--writing-content) | `Get-Content`, `Set-Content`, `Add-Content`, `Out-File`, redirection |
| [Searching & Filtering](powershell-commands.md#searching--filtering) | `Select-String`, `Where-Object`, `Select-Object`, `Sort-Object`, `Group-Object` |
| [Pipeline & Output](powershell-commands.md#pipeline--output) | `Format-Table`, `ConvertTo-Json`, `Export-Csv`, `Tee-Object`, `Write-Host` |
| [Processes & Services](powershell-commands.md#processes--services) | `Get-Process`, `Start-Process`, `Stop-Process`, service cmdlets |
| [System & Network](powershell-commands.md#system--network) | `Test-NetConnection`, `Get-NetTCPConnection`, `Resolve-DnsName`, `Get-Date` |
| [Environment & Variables](powershell-commands.md#environment--variables) | `$env:`, variables, automatic variables, quoting rules |
| [Execution Policy & Scripts](powershell-commands.md#execution-policy--scripts) | `Set-ExecutionPolicy`, running scripts, `pwsh` launch flags |
| [Packages & Modules](powershell-commands.md#packages--modules) | `Install-Module`, `Import-Module`, `winget` |
| [Web & Remoting](powershell-commands.md#web--remoting) | `Invoke-WebRequest`, `Invoke-RestMethod`, `Invoke-Command` |
| [Common Parameters](powershell-commands.md#common-parameters) | `-WhatIf`, `-Confirm`, `-Verbose`, `-ErrorAction`, preference variables |
| [Aliases](powershell-commands.md#aliases) | `ls`, `cat`, `rm`, `?`, `%` and what they map to |
| [Handy One-Liners](powershell-commands.md#handy-one-liners) | Ready-to-use commands for common tasks |

## Quick lookup

| I need to... | Go to |
|---|---|
| Find out what a command does | [`Get-Help`](powershell-commands.md#get-help) |
| Discover what properties an object has | [`Get-Member`](powershell-commands.md#get-member) |
| List files recursively by pattern | [`Get-ChildItem`](powershell-commands.md#get-childitem-aliases-ls-dir-gci) |
| Preview a delete before running it | [`-WhatIf`](powershell-commands.md#common-parameters), [`Remove-Item`](powershell-commands.md#remove-item-aliases-rm-del-rd-ri) |
| Search inside files (like `grep`) | [`Select-String`](powershell-commands.md#select-string-alias-sls) |
| Follow a log file live (like `tail -f`) | [`Get-Content -Tail -Wait`](powershell-commands.md#get-content-aliases-cat-type-gc) |
| Find what is using a port | [`Get-NetTCPConnection`](powershell-commands.md#system--network), [one-liners](powershell-commands.md#handy-one-liners) |
| Run a `.ps1` script that is blocked | [Execution policy](powershell-commands.md#execution-policy--scripts) |
| Call a REST API | [`Invoke-RestMethod`](powershell-commands.md#web--remoting) |
| Install a tool from the command line | [`winget`](powershell-commands.md#winget-windows-package-manager) |
| Make a variable permanent | [Environment & Variables](powershell-commands.md#environment--variables) |

## Tips

- Use `Get-Help <cmdlet> -Examples` for official examples without leaving the terminal.
- Add `-WhatIf` to any destructive command first to see what it would do.
- Prefer full cmdlet names in scripts; use aliases only interactively.
- Tab completion works on cmdlet names, parameter names and even parameter values.

---

Related notes: [Docker](../Docker/README.md) · [.NET](../Dotnet/README.md)
