# Eigener Windows-Runner für die CI

In der Organisation sind GitHub-gehostete Runner abgeschaltet. Die CI (`build` und `test`) läuft deshalb auf einem eigenen Windows-Runner auf dem Heimserver:

- Name `BUBBLE-WIN`, Label `bubble-win`, auf Repository-Ebene registriert.
- Ordner `C:\actions-runners\bubble` (Vererbung getrennt: SYSTEM, Administratoren, Dienstkonto).
- Dienst läuft unter seinem virtuellen Konto `NT SERVICE\actions.runner.…bubble-vscode-agent.BUBBLE-WIN`, ohne Adminrechte, ohne Kennwort.
- Node kommt über `actions/setup-node`, VS Code lädt `@vscode/test-electron` je Lauf selbst. Auf dem Server muss dafür nichts installiert sein außer Git.
- Bubble selbst läuft nicht auf dem Server, der Runner ist nur Prüfmaschine.

## Einrichtung (Admin-PowerShell auf dem Server)

1. Herunterladen und prüfen:

```powershell
& {
  $ErrorActionPreference = 'Stop'
  Get-CimInstance Win32_Service -Filter "Name LIKE 'actions.runner.%'" | Select-Object Name, State | Format-Table -AutoSize
  $dir = 'C:\actions-runners\bubble'
  if ((Test-Path $dir) -and (Get-ChildItem $dir -Force)) { throw "$dir ist nicht leer, Abbruch." }
  New-Item -ItemType Directory -Path $dir -Force | Out-Null
  icacls $dir /inheritance:r /grant:r '*S-1-5-18:(OI)(CI)F' '*S-1-5-32-544:(OI)(CI)F' /Q
  $rel = Invoke-RestMethod https://api.github.com/repos/actions/runner/releases/latest
  $asset = $rel.assets | Where-Object name -like 'actions-runner-win-x64-*.zip' | Select-Object -First 1
  $zip = Join-Path $env:TEMP $asset.name
  Invoke-WebRequest $asset.browser_download_url -OutFile $zip
  $hash = (Get-FileHash $zip -Algorithm SHA256).Hash.ToLower()
  if ("sha256:$hash" -ne $asset.digest) { throw "Prüfsumme passt nicht: $hash / $($asset.digest)" }
  Expand-Archive $zip -DestinationPath $dir
  Remove-Item $zip
  "Runner $($rel.tag_name) entpackt nach $dir"
}
```

2. Token holen: https://github.com/The-Wild-World-Organization/bubble-vscode-agent/settings/actions/runners/new?arch=x64&os=win (nur den Wert hinter `--token` kopieren, gilt eine Stunde).

3. Registrieren und auf das virtuelle Konto umstellen:

```powershell
& {
  $ErrorActionPreference = 'Stop'
  $dir = 'C:\actions-runners\bubble'
  Set-Location $dir
  $token = Read-Host 'Registrierungstoken'
  .\config.cmd --unattended --url https://github.com/The-Wild-World-Organization/bubble-vscode-agent --token $token --name BUBBLE-WIN --labels bubble-win --work _work --runasservice
  if ($LASTEXITCODE) { throw "config.cmd fehlgeschlagen ($LASTEXITCODE)" }
  $svc = (Get-Service 'actions.runner.*bubble-vscode-agent*').Name
  $acct = "NT SERVICE\$svc"
  icacls $dir /grant "${acct}:(OI)(CI)M" /T /C /Q
  sc.exe config $svc obj= $acct
  Restart-Service $svc
  Get-CimInstance Win32_Service -Filter "Name='$svc'" | Format-List Name, StartName, State
}
```

Erwartet: `StartName` = `NT SERVICE\actions.runner.…BUBBLE-WIN`, `State` = `Running`, auf GitHub unter Settings → Actions → Runners steht `BUBBLE-WIN` auf „Idle“. `sc.exe config` bewusst ohne `password=`.

## Notbremse und Rückweg

- Anhalten: `Stop-Service 'actions.runner.*bubble-vscode-agent*'` (Läufe warten dann in der Warteschlange).
- Entfernen: Token über Settings → Actions → Runners → BUBBLE-WIN → Remove holen, dann

```powershell
& { Set-Location 'C:\actions-runners\bubble'; $t = Read-Host 'Entfernungstoken'; .\config.cmd remove --token $t }
```

  und den Ordner `C:\actions-runners\bubble` löschen.

## Bekannte Grenzen

- Workflow-Code läuft auf dem Host. Das Dienstkonto hat keine Adminrechte, kann aber lesen, was „Benutzer“ lesen dürfen. Für ein privates Repository ohne fremde Mitwirkende vertretbar; strengere Trennung ginge nur mit einer eigenen Windows-VM.
- `Get-Service actions.runner.*` trifft jetzt sechs Runner.
