# The Riftbreaker Dedicated Server - Research Notes

## Confirmed Information

| Detail | Value | Source |
|--------|-------|--------|
| Game Steam App ID | 780310 | SteamDB |
| Dedicated Server App ID | 4114030 | SteamDB |
| Game Port | 6321 (UDP) | Riftbreaker Wiki / Multiplayer page |
| Anonymous SteamCMD | Yes | EXOR Studios blog |
| Headless Mode | Yes, no graphics rendering | EXOR Studios blog |
| Headless CLI param | `cli=1` | Wiki / Research |
| Config param | `config=conf\file.cfg` | Wiki / Research |
| Max Players | 4 default, up to 32 via `server_max_players_count` | Wiki / Community |
| Config directory | `Documents/The Riftbreaker/config/` | Wiki |
| Config file | `developer.cfg` (in config dir) | Wiki |
| Server Size | ~600MB (down from 11GB) | EXOR Studios blog |
| Free to Download | Yes | EXOR Studios blog |
| Wiki config tables | Empty stubs (incomplete docs) | Riftbreaker Wiki |

## SteamCMD Installation

```bash
steamcmd +login anonymous +app_update 4114030 validate +quit
```

## Needs Verification (from actual server installation)

The following details could not be confirmed from web research alone (the Riftbreaker
Fandom wiki blocks automated access). These MUST be verified by installing and running
the actual dedicated server:

1. **Executable name** - Assumed `TheRiftbreakerDedicatedServer.exe` based on naming
   conventions. May also be `riftbreaker_server.exe` or similar.

2. **Command-line arguments** - The wiki lists "server start arguments" but tables are
   empty stubs. Known/suspected parameters:
   - `cli=1` (confirmed - fully headless, no UI)
   - `config=conf\file.cfg` (confirmed - specify config file)
   - `-server_name` / `-max_players` / `-port` / `-password` / `-visibility` (suspected)
   - `-no_steam` (confirmed from blog - enables non-DRM direct IP mode)
   - `-save_name` (suspected - for loading specific save files)

3. **Configuration files** - Partially known:
   - `developer.cfg` in `Documents/The Riftbreaker/config/` directory
   - Console command `set server_max_players_count "[Number]"` to change max players
   - The game uses Schmetterling engine (EXOR proprietary); config format to be confirmed

4. **Console output patterns** - Need to observe actual server logs to create accurate:
   - `AppReadyRegex` (server started pattern)
   - `UserJoinRegex` (player connected pattern)
   - `UserLeaveRegex` (player disconnected pattern)

5. **Linux support** - EXOR Studios stated Linux is "not officially supported" for the
   dedicated server. Proton via SteamCMD may work (template includes Proton config).

6. **Steam Query Port** - Assumed 27015 (standard). May be different.

## Next Steps

1. Install the dedicated server via SteamCMD on the Proxmox test VM
2. Identify the actual executable name
3. Run the server and capture console output for regex patterns
4. Document all command-line arguments (run with `-help` or `--help`)
5. Find configuration files in the installation directory
6. Test Proton compatibility on Linux
7. Update the AMP template with verified information

## Sources

- [EXOR Studios - Dedicated Server Progress Report](https://www.exorstudios.com/blog/rb-dedicated-server)
- [EXOR Studios - Co-Op Update Launch](https://www.exorstudios.com/blog/co-op-update-launch)
- [SteamDB - App 4114030](https://steamdb.info/app/4114030/config/)
- [Riftbreaker Wiki - Dedicated server configuration](https://riftbreaker.fandom.com/wiki/Dedicated_server_configuration) (blocked, needs manual access)
- [Riftbreaker Wiki - Multiplayer](https://riftbreaker.fandom.com/wiki/Multiplayer)
