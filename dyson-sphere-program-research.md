# Research notes — Dyson Sphere Program + Nebula AMP template

Background facts gathered while building the template. See `dyson-sphere-program-design.md`
for the design and `VERSIONS.md` for the pinned versions.

## The game
- **Dyson Sphere Program**, Steam app `1366540`. Windows-only Unity (**Mono**) game.
- **Paid title, no anonymous SteamCMD build** → installing needs Steam credentials for an
  account that owns DSP. `App.SteamUpdateAnonymousLogin=False`, `App.SteamForceLoginPrompt=True`.
- **No official dedicated server.** Multiplayer is entirely the third-party Nebula mod.

## Nebula
- BepInEx plugin. Repo: `NebulaModTeam/nebula` (the active fork; `hubastard/nebula` is old).
- **Headless mode is first-class:** `DSPGAME.exe -batchmode -nographics -nebula-server -load-latest`.
  Other args: `-load "<name>"`, `-newgame <seed> <stars> <resmult>`, `-ups <5..240>`.
- **Port: TCP 8469** (WebSocket; **no UDP**). cfg key `HostPort`.
- Config file `BepInEx/config/nebula.cfg`, single section `[Nebula - Settings]`, keys
  (verbatim from source `MultiplayerOptions.cs` / `Config.cs`):
  `ServerPassword`, `HostPort` (8469), `RemoteAccessEnabled` (false),
  `RemoteAccessPassword`, `AutoPauseEnabled` (true).
- Remote admin chat commands (when `RemoteAccessEnabled=true`): `/server login <pw>`,
  `/server save`, `/server load`. Use these to save before restart.
- **Nebula 0.9.22 dependencies** (Thunderstore): `xiaoye97-BepInEx-5.4.17`,
  `nebula-NebulaMultiplayerModApi-2.1.0`, `PhantomGamers-IlLine-1.0.0`,
  `starfi5h-ErrorAnalyzer-1.3.3`, `starfi5h-NebulaCompatibilityAssist-0.5.0`,
  `starfi5h-BulletTime-1.5.13`.

## Running on Linux (Wine + Goldberg)
- DSP is a Windows exe → run under Wine. Official Unity-on-Wine AMP templates (ASKA,
  Sunkenland) use `App.ExecutableLinux=/usr/bin/xvfb-run` + `-a wine "./Game.exe"` with
  `-batchmode -nographics`. We mirror that.
- **BepInEx/Doorstop injection under Wine** needs `WINEDLLOVERRIDES=winhttp.dll=n,b`
  (load native winhttp.dll before Wine's builtin). Without it BepInEx never loads.
- **Goldberg / gbe_fork** (`Detanup01/gbe_fork`) replaces `steam_api64.dll` so the game
  runs with no live Steam login. Use the **regular** build (`release/regular/x64/`); the
  **experimental** build blocks non-LAN IPs and would prevent internet play.
- `.7z` asset → needs p7zip (`Meta.ExtraContainerPackages=["p7zip-full"]`, extracted with
  `7z` in a bash stage).
- BepInEx zip (`xiaoye97-BepInEx`) is the Valheim-style wrapped pack: a top-level
  `BepInExPack/` folder whose contents (`BepInEx/`, `winhttp.dll`, `doorstop_config.ini`)
  must be copied up into the game root.
- BepInEx writes `BepInEx/LogOutput.log` by default → we tail that (`AdminMethod=TailLogFile`)
  rather than relying on stdout, which is unreliable under Wine.

## Sources
- Official templates: `CubeCoders/AMPTemplates` (aska, sunkenland, abiotic-factor, valheim).
- Nebula: `NebulaModTeam/nebula` (wiki Setup-Headless-Server, Hosting-and-Joining; source
  `MultiplayerOptions.cs`, `Config.cs`, `Dedicated_Server_Patches.cs`).
- Thunderstore API for version pins; `Detanup01/gbe_fork` releases for Goldberg.
- Community references: `WhyKickAmooCow/dsp-docker-server`, `AlienXAXS/DSPNebulaDocker`.
