# Pinned Versions — Dyson Sphere Program + Nebula template

This file is the **single source of truth** for the versions the DSP template installs.
Everything here is also exposed as an editable field in the AMP **Configuration** tab,
so you can bump a version either by editing the `DefaultValue` in
`dysonsphereprogramconfig.json` (affects new instances) **or** by overriding the field
on an existing instance and re-running **Update**.

## Current pins

| Component | Version field (config) | Default | Source / download |
|-----------|------------------------|---------|-------------------|
| Dyson Sphere Program | `DSPBranch` | `public` | SteamCMD app `1366540`, Windows build |
| Goldberg (gbe_fork) | `GoldbergTag` | `release-2026_05_30` | `https://github.com/Detanup01/gbe_fork/releases/download/<tag>/emu-win-release.7z` (uses `release/regular/x64/steam_api64.dll`) |
| BepInEx (xiaoye97) | `BepInExVersion` | `5.4.17` | `https://gcdn.thunderstore.io/live/repository/packages/xiaoye97-BepInEx-<ver>.zip` (BepInEx 5.x Mono) |
| Nebula API | `NebulaApiVersion` | `2.1.0` | `https://gcdn.thunderstore.io/live/repository/packages/nebula-NebulaMultiplayerModApi-<ver>.zip` |
| Nebula mod | `NebulaVersion` | `0.9.22` | `https://gcdn.thunderstore.io/live/repository/packages/nebula-NebulaMultiplayerMod-<ver>.zip` |

### Fixed sub-dependencies (edit in `dysonsphereprogramupdates.json`)

Nebula `0.9.22` declares these as dependencies. They are pinned directly in
`dysonsphereprogramupdates.json` (not exposed as config fields to avoid clutter).
When you bump Nebula, check its Thunderstore dependency list and update these to match:

| Dependency | Pinned version | Thunderstore full-name |
|------------|----------------|------------------------|
| IlLine | `1.0.0` | `PhantomGamers-IlLine` |
| ErrorAnalyzer | `1.3.3` | `starfi5h-ErrorAnalyzer` |
| NebulaCompatibilityAssist | `0.5.0` | `starfi5h-NebulaCompatibilityAssist` |
| BulletTime | `1.5.13` | `starfi5h-BulletTime` |

## The lockstep rule (read before bumping)

**Nebula must match the DSP game build, and every player must run the exact same
Nebula version as the server.** If DSP updates past the Nebula release, multiplayer
breaks until Nebula catches up.

Recommended flow when a new Nebula release lands:
1. Check the new Nebula version's Thunderstore page for its required **BepInEx** and
   **NebulaApi** versions and its **dependency list**.
2. Update `NebulaVersion`, `NebulaApiVersion`, `BepInExVersion` (defaults in
   `dysonsphereprogramconfig.json`) and the sub-dependency versions in
   `dysonsphereprogramupdates.json`.
3. Update the table above and commit.
4. Tell players to update their client mods to the same versions.
5. Re-run **Update** on the instance.

## How to bump (quick)

- **One instance, quick test:** Configuration tab → change the version field → Update.
- **Permanent / all new instances:** edit the `DefaultValue` in
  `dysonsphereprogramconfig.json` (and sub-deps in `dysonsphereprogramupdates.json`),
  update the table above, commit + push. AMP re-pulls the repo on the next deploy.

## How to verify a download URL before committing a bump

```
curl -sIL "https://gcdn.thunderstore.io/live/repository/packages/nebula-NebulaMultiplayerMod-<ver>.zip" | head
# expect: HTTP/.. 200  and  content-type: application/zip
```

## Items that still need a live first-run check

These could not be confirmed without running the server once on the AMP host:

1. **`Console.AppReadyRegex`** in `dysonsphereprogram.kvp` is a best-guess for the
   Nebula "server ready" log line. After the first start, open
   `dysonsphereprogram/1366540/BepInEx/LogOutput.log`, find the actual line Nebula
   prints when the headless server is listening, and tighten the regex to match it.
   Until then the instance may sit on "Starting" or flip to "Running" early.
2. **Player join/leave regexes** are best-guesses — confirm against `LogOutput.log`.
3. **`nebula.cfg` keys** (`ServerPassword`, `HostPort`, `RemoteAccessEnabled`,
   `RemoteAccessPassword`, `AutoPauseEnabled`, section `[Nebula - Settings]`) were taken
   from the Nebula source. Confirm the generated file matches after first run; adjust
   `dysonsphereprogrammetaconfig.json` / the seed file if Nebula changes them.
