# Design – Dyson Sphere Program (Nebula) AMP Template

**Author:** Cronus (with Claude)
**Date:** 2026-06-05
**Repo:** CronusAMPTemplates
**Template prefix:** `dysonsphereprogram`

## Purpose

Add a custom AMP (CubeCoders Application Management Panel) Generic-module template to
the `CronusAMPTemplates` git repo that lets us host a **Dyson Sphere Program (DSP)
multiplayer server** via the **Nebula** mod's headless server mode, on our Linux AMP
host. AMP pulls templates from this repo (added under Configuration → Instance
Deployment as `Cronus87/CronusAMPTemplates:main`).

DSP has **no official dedicated server**; multiplayer is provided entirely by the
third-party **Nebula** mod (BepInEx plugin), which exposes a first-class headless mode:

```
DSPGAME.exe -batchmode -nographics -nebula-server -load-latest
```

## Key facts (from research)

| Item | Value |
|------|-------|
| DSP Steam App ID | `1366540` (paid title, **no anonymous SteamCMD build**) |
| Runtime | Windows-only Unity (Mono) — runs on Linux via **Wine** |
| Mod loader | **BepInEx** (Doorstop proxies `winhttp.dll` on Windows) |
| Multiplayer mod | **Nebula** + **NebulaAPI** (Thunderstore: `nebula/NebulaMultiplayerMod`, `nebula/NebulaMultiplayerModApi`) |
| Default port | **TCP 8469** (WebSocket; no UDP) |
| Nebula config | `BepInEx/config/nebula.cfg` (ini) |
| BepInEx console | enable `[Logging.Console] Enabled = true` in `BepInEx/config/BepInEx.cfg`; `LogOutput.log` always written |
| Headless | `-batchmode -nographics`; no GPU required |

## Decisions (locked with user)

1. **Game acquisition:** SteamCMD download using a Steam account that **owns DSP**
   (credentials entered in AMP config), then run under the **Goldberg Steam emulator**
   so no live Steam login is needed at runtime. (Mirrors `WhyKickAmooCow/dsp-docker-server`.)
2. **Mod install:** Auto-fetch **pinned versions** of BepInEx + Nebula from Thunderstore.
   Versions are exposed as AMP settings (defaults = pinned) **and** documented in
   `VERSIONS.md` so bumping is a one-line edit. DSP build and Nebula version are kept in
   **lockstep**.
3. **Base pattern:** Official **Unity-on-Wine** templates (ASKA / Sunkenland), not our
   earlier unverified Riftbreaker draft. Goldberg is the only necessary deviation from
   official practice (no official template covers paid titles).

## Architecture / template files

AMP auto-discovers `.kvp` files; `manifest.json` (repo-level) stays unchanged.

| File | Purpose |
|------|---------|
| `dysonsphereprogram.kvp` | Core Generic-module config (meta, launch, console, ports refs) |
| `dysonsphereprogramconfig.json` | Settings UI (server + version-pin fields) |
| `dysonsphereprogrammetaconfig.json` | Maps settings → `nebula.cfg` (ini) |
| `dysonsphereprogramports.json` | TCP 8469 |
| `dysonsphereprogramupdates.json` | Install pipeline (SteamCMD → Goldberg → BepInEx → Nebula) |
| `dyson-sphere-program-research.md` | Research notes (sources, gotchas) |
| `VERSIONS.md` | Pinned-version table + how-to-bump procedure |

### Launch (`.kvp`) — modelled on ASKA / Sunkenland

```
App.ExecutableLinux=/usr/bin/xvfb-run
App.LinuxCommandLineArgs=-a wine "./DSPGAME.exe"
App.CommandLineArgs={{$PlatformArgs}} -batchmode -nographics -nebula-server {{WorldSourceArgs}} -logFile "./BepInEx/LogOutput.log"
App.EnvironmentVariables={"SteamAppId":"1366540","WINEPREFIX":"{{$FullRootDir}}.wine","WINEARCH":"win64","WINEDEBUG":"-all","WINEDLLOVERRIDES":"winhttp.dll=n,b"}
App.MonitorChildProcessName=^.*DSPGAME\.exe.*$
```

- `{{WorldSourceArgs}}` is computed from the **World Source** setting (see below).
- `WINEDLLOVERRIDES=winhttp.dll=n,b` makes Wine load BepInEx's native `winhttp.dll`
  before its builtin — without this BepInEx never initialises and Nebula never loads
  (official equivalent: Abiotic Factor's `version.dll=n,b` for UE4SS).
- `xvfb-run -a` allocates a virtual display automatically; no `DISPLAY` var needed
  (matches ASKA / Sunkenland, both Unity-on-Wine).

### Console / readiness — Sunkenland `TailLogFile` pattern

```
App.HasReadableConsole=True
App.HasWriteableConsole=False
App.AdminMethod=TailLogFile
App.TailLogFilePath={{$FullBaseDir}}BepInEx/LogOutput.log
App.ApplicationReadyMode=RegexMatch   (or ServerInfo)
App.ExitMethod=OS_CLOSE
```

Rationale: community reports confirm BepInEx output does not reliably reach stdout under
Wine, but BepInEx **always** writes `BepInEx/LogOutput.log`. We tail that file.

> **OPEN — needs live verification:** the exact "server ready" log line Nebula writes is
> not published. `Console.AppReadyRegex` will be set to a best guess and **must be
> confirmed against a real first-run `LogOutput.log`**. Until confirmed the instance may
> show "Starting" indefinitely or flip ready early. This is the single item that cannot
> be finalised without a live run on the AMP host.

### Install pipeline (`updates.json`) — ordered stages

1. **SteamCMD download DSP** — `UpdateSource: SteamCMD`, `UpdateSourceData: 1366540`,
   `UpdateSourceArgs: 1366540`, `ForceDownloadPlatform: Windows`. Credentialed
   (`App.SteamUpdateAnonymousLogin=False`, `App.SteamForceLoginPrompt=True`).
   `UpdateSourceVersion` = `{{DSPBranch}}` for build pinning.
2. **Create `steam_appid.txt`** — `UpdateSource: CreateFile`, contents `1366540`, in game dir.
3. **Initialise Wine prefix** — `Executable` `/bin/bash` running
   `WINEPREFIX=… WINEARCH=win64 WINEDEBUG=-all /usr/bin/wineboot --init` (official form).
4. **Install Goldberg emulator** (pinned) — fetch `{{GoldbergVersion}}` release, place
   `steam_api64.dll` + `steam_settings/` (with `steam_appid.txt`=1366540) in the game dir.
   *Only deviation from official practice.* Gated by a `UseGoldberg` setting (default true).
5. **Install BepInEx** (pinned `{{BepInExVersion}}`) — `FetchURL` Thunderstore zip →
   `ExtractArchive` to game root (provides `winhttp.dll` Doorstop proxy).
6. **Install NebulaAPI + Nebula** (pinned `{{NebulaVersion}}`) — `FetchURL` Thunderstore
   zips → `ExtractArchive` → copy DLLs into `BepInEx/plugins/`.
7. **Enable BepInEx console logging** — `sed` flip `[Logging.Console] Enabled = false`
   → `true` in `BepInEx/config/BepInEx.cfg` (like ASKA's render-config edit).

All fetch URLs use explicit pinned versions (no "latest") so installs are reproducible.

### Settings (`config.json`) → mapped via `metaconfig.json` to `nebula.cfg` (ini)

Server settings:
- **Server Password** → `nebula.cfg` `ServerPassword`
- **Auto-Pause when empty** → `AutoPauseEnabled` (default `true`)
- **Remote Admin** enable + password → `RemoteAccessEnabled` / `RemoteAccessPassword`
  (lets us issue `/server save` before restarts)
- **Game speed (UPS)** → launch arg `-ups <n>` (range 5–240)
- **Max Players** → `$MaxUsers` (AMP-level, default 4)

**World Source** (enum) drives `{{WorldSourceArgs}}`:
- `Load latest save` → `-load-latest` (default)
- `Load named save` → `-load "<SaveName>"` (text field)
- `New game` → `-newgame <Seed> <StarCount> <ResourceMultiplier>` (three fields)

Version-pin fields (defaults = pinned versions in `VERSIONS.md`):
- `DSPBranch`, `BepInExVersion`, `NebulaVersion`, `GoldbergVersion`, `UseGoldberg`

### Ports (`ports.json`)

```json
[
  { "Protocol": "TCP", "Port": 8469, "Ref": "GamePort",
    "Name": "Game Port", "Description": "Nebula multiplayer (WebSocket/TCP)" }
]
```
`App.PrimaryApplicationPortRef=GamePort`. No Steam query port (Nebula is direct-connect,
not server-browser).

### `VERSIONS.md` (the update system)

A single table: DSP build ID, Goldberg version, BepInEx version, NebulaAPI version,
Nebula version, with the matching Thunderstore/GitHub URLs and the DSP version each
Nebula release targets. "How to bump" section: edit the default in `config.json`
(and/or the URL in `updates.json`), commit, push; AMP re-pulls on next instance
update. Records the **lockstep rule**: never bump DSP past a Nebula release, and all
clients must run the identical Nebula version.

## Known caveats / risks

| Risk | Handling |
|------|----------|
| Steam credentials stored on AMP host | Accepted (unavoidable for paid-game SteamCMD). Documented. |
| Nebula "server ready" regex unknown | Best-guess in template; **must verify on live first run**. Fallback: `ServerInfo` mode. |
| Clean-shutdown autosave unreliable under Wine | `ExitMethod=OS_CLOSE` + enable Nebula autosave + document "use Remote Admin `/server save` before restart". Possible future one-click Save UserAction. |
| BepInEx logs may not reach stdout under Wine | Use `TailLogFile` on `LogOutput.log` instead of STDIO. |
| DSP↔Nebula version drift breaks joins | Pinned versions + lockstep rule in `VERSIONS.md`. |
| Not eligible for official CubeCoders repo | Intentional — personal-use template (paid game requires login). |

## Out of scope (YAGNI)

- Native Linux build (DSP has none).
- Server-browser listing (Nebula is direct-connect).
- One-click in-panel save button (possible later UserAction).
- Windows AMP-host support beyond what falls out for free (target is our Linux host).

## Acceptance (high level)

1. Repo contains the seven files; `manifest.json` unchanged; valid JSON/KVP.
2. AMP lists "Dyson Sphere Program" after pulling the repo.
3. Fresh instance: install pipeline completes (SteamCMD w/ creds → Goldberg → BepInEx →
   Nebula) with no fatal stage.
4. Instance starts; `LogOutput.log` shows BepInEx + Nebula loaded; port 8469 listening.
5. A DSP client (matching Nebula version) connects to `host:8469`.
6. `AppReadyRegex` confirmed against the real ready line; instance shows "Running".
