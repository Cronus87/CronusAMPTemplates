# DSP + Nebula AMP Template — Troubleshooting & Lessons Learned

This document captures every problem hit while building the CubeCoders AMP "Generic" module template that hosts a **Dyson Sphere Program (DSP)** multiplayer server via the **Nebula** mod — headless, under Wine, on a Linux AMP host. Each issue records the symptom, root cause, the fix that worked, anything tried that did **not** work, and additional notes, so the same mistakes aren't repeated.

Australian English spelling is used throughout.

---

## Environment

- **AMP web panel:** http://192.168.1.25:8080. AMP version **2.7.2.8**.
- **Host:** Proxmox LXC container **CT102** "AMP-Game-Servers" on node **pm**. Enter it from the Proxmox host as:

  ```bash
  sudo /usr/sbin/pct exec 102 -- bash -c '...'
  ```

  The Proxmox MCP runs commands on the PVE host as user `mcp-admin`; `pct` needs the full path **plus** `sudo`.
- **AMP runs as user `amp`.** Datastore at `/home/amp/.ampdata/instances/ADS01`. Logs in `ADS01/AMP_Logs/AMPLOG_*.log`.
- **IMPORTANT — Wine and xvfb are NOT installed on the CT102 host.** The game runs inside a Docker container `AMP_DysonSphereProgram01` using image `cubecoders/ampbase:wine-stable`. The instance directory is bind-mounted to `/AMP` inside that container. So `wine`/`xvfb-run` only exist **inside the container**, and game file paths in logs appear as `/AMP/dysonsphereprogram/1366540/...` or `Z:\AMP\...`.
- **The DSP instance:** name **DysonSphereProgram01**, InstanceID `bbaa278a-2591-47a5-b6eb-7b0573a017a2`. AMP assigned it game port **25624** (NOT the template's default 8469 — AMP auto-assigns ports).
- **DSP Steam appid `1366540`** (paid; no anonymous SteamCMD download).
- **Mod stack:**
  - BepInEx **5.4.17** (xiaoye97, Mono).
  - Nebula **0.9.22**, dependencies:
    - nebula-NebulaMultiplayerModApi **2.1.0**
    - PhantomGamers-IlLine **1.0.0**
    - starfi5h-ErrorAnalyzer **1.3.3**
    - starfi5h-NebulaCompatibilityAssist **0.5.0**
    - starfi5h-BulletTime **1.5.13**
  - Goldberg = **Detanup01/gbe_fork release-2026_05_30**, regular/x64 `steam_api64.dll`.
- **Useful AMP API** (auth: `POST /API/Core/Login` with headers `Content-Type: application/json` **AND** `Accept: application/json`, body `{username,password,token:"",rememberMe:false}` → `sessionID`; the discord-bot account has admin `"*"`):
  - `ADSModule/GetSupportedApplications` — lists creatable apps; our template shows as "**Cronus / Dyson Sphere Program**".
  - `ADSModule/RefreshRemoteConfigStores` `{force:true}` — git-pull repos into ADS memory.
  - `ADSModule/RefreshInstanceConfig` `{InstanceId}` — regenerate an instance's baked config.
  - `ADSModule/StartInstance` `{InstanceName}`.
  - **Note:** `GetDeploymentTemplates` returns 0 — it is a **different** concept (saved instance bundles). Do **not** use it to check config templates.

---

## Issue 1: Custom template repo never appears in AMP's New Instance list

- **Symptom:** Neither the new DSP template nor an older Riftbreaker template ever showed up when creating an instance, despite the repo being added under **Configuration → Instance Deployment → Configuration Repositories** and fetched.
- **Root cause:** The repo's root `manifest.json` had an **EMPTY** `"prefix": ""`. AMP requires a custom repo's manifest to have a **unique, non-empty** prefix (the docs say `id`, `origin`, `url`, **and** `prefix` must be unique to your setup). With an empty prefix nothing surfaces and no error is logged.
- **Fix that worked:** Set `"prefix": "Cronus"` (and a fresh unique `id`) in `manifest.json`. Templates then appear in the app list named "Cronus / <DisplayName>" (filed under the prefix — e.g. search "Cronus" or "Dyson").
- **Also required:** Each template's `.kvp` needs `Meta.AppConfigId` (a GUID) — we had this.
- **Tried but did NOT work:** Clicking Fetch + browser refresh alone; restarting ADS01 alone. Those steps are necessary later but do nothing while the prefix is empty.
- **Notes:** To verify server-side (bypasses browser cache), call `ADSModule/GetSupportedApplications` and look for the `FriendlyName`.

---

## Issue 2: Template still not visible after the prefix fix

- **Symptom:** Still not in the list right after fixing the prefix.
- **Root cause:** AMP caches the application list (a) in ADS memory and (b) client-side in the browser.
- **Fix that worked:** After Fetch, **restart the ADS01 instance** (or call `RefreshRemoteConfigStores`), **and** hard-refresh the browser (Ctrl+F5) / use an incognito window.
- **Notes:** Both caches must be cleared; clearing only one leaves the template hidden.

---

## Issue 3: BepInEx/Nebula install stages fail — "Could not find a part of the path"

- **Symptom:** Install log shows e.g. `Unable to move /tmp/xxxx to /AMP/dysonsphereprogram/1366540/_bepinex_tmp/bepinex.zip : Could not find a part of the path`, then the whole pipeline aborts; BepInEx and all mods never install.
- **Root cause:** AMP's `FetchURL` update source downloads to `/tmp` and then **moves** the file into `UpdateSourceTarget`, but it does **not** create the target directory first.
- **Fix that worked:** Add `mkdir -p` stages (an Executable/bash stage, or `CreateDirectory`) **before** every `FetchURL` that targets a not-yet-existing directory — i.e. before the BepInEx fetch (creates `_bepinex_tmp`) and before the Nebula/plugin fetches (creates each `BepInEx/plugins/<modname>` directory).
- **Notes:** This applies to every mod plugin directory, not just BepInEx.

---

## Issue 4: Fixing the git repo did not change an already-created instance

- **Symptom:** After editing `updates.json` in git and even `git reset`-ing the repo copy on the host, re-running Update on the existing instance still ran the **old** pipeline.
- **Root cause:** At instance creation AMP **bakes** the resolved template into the instance's own `GenericModule.kvp` (`App.UpdateSources` is stored there as AMP's internal serialized JSON with numeric enums; the instance does **not** keep a separate `updates.json` and does **not** read the repo copy live for that). ADS also caches templates in memory.
- **Fix that worked:** `ADSModule/RefreshRemoteConfigStores` `{force:true}` to reload the (already git-updated) repo into ADS memory, then `ADSModule/RefreshInstanceConfig` `{InstanceId}` to regenerate the instance's settings manifest and `App.UpdateSources` from the refreshed template.
- **IMPORTANT caveat (tried, partially did NOT work):** `RefreshInstanceConfig` updates the settings manifest and `App.UpdateSources`, but does **not** rewrite `App.CommandLineArgs` (or other core `App.*` fields). To change the launch command on an existing instance you must edit the instance's `GenericModule.kvp` directly (as user amp: `sudo -u amp sed -i ...`) or recreate the instance.
- **Notes:** When in doubt about core `App.*` fields, recreating the instance is the cleanest path.

---

## Issue 5: SteamCMD stuck "waiting for Steam Guard", no code arrives

- **Symptom:** Install log: `This account is protected by a Steam Guard mobile authenticator` then `Steam Guard needed!` / `Please confirm the login in the Steam Mobile app`. No email or code arrives.
- **Root cause:** The Steam account uses the mobile authenticator's **device-approval** flow, not an emailed code.
- **Fix that worked:** Re-run Update, then open the **Steam Mobile app** and tap **Approve** on the sign-in prompt (bring the app to the foreground if no push arrives; check **Confirmations**). After approving once, SteamCMD caches credentials, so later updates log in without prompting.
- **Notes:** DSP is a paid title — anonymous SteamCMD does **not** work; the account must own DSP. `App.SteamUpdateAnonymousLogin=False`, `App.SteamForceLoginPrompt=True`.

---

## Issue 6: World selection argument `-newgame 0 64 1` (and `-load MySave`) ignored

- **Symptom:** Setting World/Save to `-newgame 0 64 1` did not start a new game — the game instead booted toward the normal menu and quit; Nebula never logged "Initializing dedicated server". `-load-latest` (no spaces) **did** engage Nebula.
- **Root cause:** AMP passes a single setting **value** that contains spaces as **one** argument (argv element). So Nebula never sees a bare `-newgame` token; the same problem would hit `-load MySave`.
- **Fix that worked:**
  - (a) For a new world use Nebula's single-token `-newgame-cfg` (reads/creates `BepInEx/config/nebulaGameDescSettings.cfg`).
  - (b) For choosing a save, split into **separate** settings/tokens: a "World Mode" enum (values `-load-latest` / `-load` / `-newgame-cfg`) plus a separate "Save Name" text field, composed in `CommandLineArgs` as `{{WorldMode}} {{SaveName}}` so the flag and the name arrive as separate argv elements.
- **Tried but did NOT work:** `-newgame 0 64 1` in one field; `-load MySave` in one field.
- **Notes:** Any multi-token DSP/Nebula flag must be split across separate settings to survive argv passing.

---

## Issue 7: Goldberg "SteamAPI_Init() failed" on a full game boot — blocks server-side new game

- **Symptom:** When the game does a **full** boot (which creating a brand-new galaxy requires), the log shows `[Steamworks.NET] SteamAPI_Init() failed`, then `UIRunner.HandleApplicationQuit()` and several `NullReferenceException`s during teardown, exit code 0.
- **Root cause:** **NOT fully resolved.** The Goldberg/gbe_fork `steam_api64.dll` fails to initialise under Wine during a full boot. Leading hypotheses: the Wine prefix is missing the MSVC runtime the Goldberg DLL needs (try `winetricks vcrun2019` into the prefix), and/or Goldberg needs `steam_settings/steam_interfaces.txt` generated from the original `steam_api64.dll`. **STILL OPEN.**
- **Key insight / current workaround:** Nebula's headless **load** path (`-load-latest` / `-load <name>`) intercepts very **early** (logs "Initializing dedicated server" before the Unity/shader boot) and is **Steam-independent** — it does not call `SteamAPI_Init`. So **loading** an existing save works; only **new-game-on-server** is blocked. Workaround: build the world on a PC DSP client, save it, upload the `.dsv` to the server, then use Load latest/specific.
- **Notes:** Until the Goldberg init is fixed under Wine, server-side new galaxy creation is not possible — seed worlds from a desktop client.

---

## Issue 8: Save folder not visible in AMP File Manager

- **Symptom:** Cannot find the DSP Save folder anywhere in File Manager.
- **Root cause:** DSP saves live inside the Wine prefix at `dysonsphereprogram/.wine/drive_c/users/amp/Documents/Dyson Sphere Program/Save/`, and AMP's File Manager **hides dot-folders** (`.wine`).
- **Fix that worked:** Create a visible symlink `dysonsphereprogram/Save` → the hidden prefix Save folder (added as an install stage; also created manually on the existing instance with `sudo -u amp ln -sfn ...`). Upload/swap saves via `dysonsphereprogram/Save` in File Manager.
- **Notes:** Alternative — enable a "show hidden files" toggle if the File Manager has one.

---

## Issue 9: Could not manually run the game under Wine for testing

- **Symptom:** Trying to launch `DSPGAME.exe` manually (to test args / generate a save) failed with `wine: '/AMP/dysonsphereprogram/.wine' is not owned by you`.
- **Tried but did NOT work:**
  - `docker exec -u root` (Wine refuses a prefix it does not own).
  - `docker exec -u 1000` and `docker exec -u amp` (still "not owned by you" — likely a nested LXC+Docker uid-mapping quirk).
  - Running on the bare CT102 host also fails because Wine/xvfb are not installed there.
- **Lesson:** Don't try to reproduce by manually invoking Wine outside AMP; AMP's own launcher sets up the environment correctly. Use AMP (`StartInstance`) to launch and watch the logs instead.

---

## Issue 10 (OPEN): Server marked "failed to start" even though the save loads

- **Symptom:** With a save uploaded and `-load-latest`, one run reached "Loading latest save: server" and patched headless successfully, but AMP still reported "Application failed to start 2 times". Retries alternate between the good headless-load path and a full-boot crash.
- **Suspected cause (under investigation):** `Console.AppReadyRegex` in the `.kvp` is a best-guess and likely does not match Nebula's real "server ready" log line, so AMP never sees the server as ready. Also the alternation between headless and full-boot paths needs understanding.
- **Status:** **OPEN.** Next step: capture the real ready/listening log line from a clean run and set `Console.AppReadyRegex` accordingly.

---

## RESOLVED — what finally made it work (2026-06-06)

The server now runs and clients can join. The fixes below resolved the previously
open issues (7, 10, and the join failure) and are now baked into the template files.

1. **Steam init (resolves Issue 7).** Use the **ORIGINAL Mr_Goldberg** `steam_api64.dll`
   — **NOT gbe_fork**, which needs an MSVC runtime Wine lacks. Install it to
   `DSPGAME_Data/Plugins/x86_64/steam_api64.dll` (the copy Unity actually loads) —
   **NOT the game root**. Remove any game-root `steam_api64.dll`: it shadows the
   correct one via the Windows `LoadLibrary` search order. Add
   `steam_settings/disable_networking.txt`. Result: `SteamAPI_Init` succeeds
   ("Steam achievement data sync successful"). The DLL URL is now the `GoldbergUrl`
   config field (default = a known-good GitLab CI artifact).
2. **BepInEx mod-loading race under Wine** (game hung at the menu with no GPU): greatly
   improved by the community Wine env — `WINEDLLOVERRIDES="mscoree=n,b;mshtml=n,b;winhttp=n,b"`
   — and a proper virtual display: `xvfb-run -a -s "-screen 0 1280x720x24"`.
   **NOT 100% deterministic, though.** BepInEx's preloader patches `UnityEngine.Application..cctor`
   to inject the chainloader (which loads the mods). A static ctor runs once; under Wine the
   timing of that patch vs the game's first access to `Application` varies per start. Win the
   race -> mods load -> Nebula runs. Lose it -> only the *preloader* runs (zero plugins), the
   game boots vanilla and HANGS at `ApplyPowerSettings` (high CPU, AMP stuck on "Starting",
   no "Game loaded" in the log).
   **WORKAROUND - "the re-roll":** Stop the instance and Start it again. It usually catches a
   good run within a try or two. A good run logs the mods loading + "Game loaded" and AMP
   shows "Running". (A deterministic fix would need BepInEx entrypoint tuning or a
   start-watchdog that auto-restarts on a hang - not yet implemented.)
3. **AMP stuck on "Starting" (resolves Issue 10).** `Console.AppReadyRegex` must match a
   line the server actually prints; **"Game loaded"** is reliable. Set:
   `Console.AppReadyRegex=^.*(Game loaded|Initializing dedicated server|Loading latest save).*$`
4. **Client could not join ("Server is missing mod NebulaCompatibilityAssist").** The
   server mod set must **EXACTLY** match the client's r2modman set.
   NebulaCompatibilityAssist needs **DSPModSave** (`CommonAPI-DSPModSave` **1.2.2**) to
   load; the client also used **NebulaCompatibilityAssist 0.5.1** (not 0.5.0) and did
   **NOT** have CommonAPI core. Match versions exactly and don't add extra mods.

## Still open / TODO

1. **Server-side new-game now WORKS** (confirmed 2026-06-06, once the Steam fix was in).
   World Mode "New game" (`-newgame-cfg`) generates a fresh galaxy on the server (creates
   `BepInEx/config/nebulaGameDescSettings.cfg`, logs `Generate Terrain/Vegetables/Veins`),
   no save upload needed. Uploading a PC-made save is only needed for a *specific* existing
   world. (Subject to the same start-race as everything else — re-roll if it hangs.)
0. **Startup reliability (the re-roll)** is the main remaining rough edge — see RESOLVED
   item 2. If a start hangs on "Starting", Stop/Start to re-roll. A deterministic fix
   (entrypoint tuning or a start-watchdog) is a possible future improvement.
2. **Decide whether to silence** the harmless Discord RPC warning.
3. **Note:** the **gbe_fork** approach and **p7zip** are **no longer used** (we no longer
   download a `.7z`). Connecting via a **domain** (e.g. `host:25624` through a DNS name)
   is a separate DNS / port-forwarding matter, **not** a server config issue.
