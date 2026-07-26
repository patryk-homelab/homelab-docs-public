# Mac mini Homelab Inventory

Last updated: 2026-07-26

This file documents the current Mac mini backup/control homelab setup. It is documentation only and should stay conservative. The Mac mini is intentionally lightweight to preserve RAM and disk for local LLM/OpenLLM work.

## Purpose

- Mac mini role: backup/control homelab node.
- Primary homelab remains the Fedora miniPC at `192.168.10.12`.
- This Mac mini should remain minimal and avoid unnecessary background services.
- Prefer lightweight native services over Docker Desktop or heavier management layers.
- Current known backup/control role includes:
  - native AdGuard Home backup DNS
  - SSH access
  - Screen Sharing access
  - Tailscale connectivity
  - native MeTube service
  - native UpSnap Wake-on-LAN management
  - wake/recovery target for remote access continuity
- Full-system Time Machine and the separate homelab service/configuration backup are implemented.

## Host identity

- User: `patrykmac`
- Current hostname: `Patryks-Mac-mini.local`
- Current macOS version: `26.5.2` (`25F84`)
- Architecture: `arm64`

## Network

- Router LAN IP: `192.168.10.1`
- Fedora miniPC: `192.168.10.12`
- Mac mini LAN IP: `192.168.10.13`
- LAN subnet: `192.168.10.0/24`
- Ethernet interface: `en0`
- Ethernet MAC: `14:98:77:6b:d2:f9`
- SSH alias from MacBook: `ssh macmini`
- Router DNS should remain unchanged during unrelated documentation work.

## Time

- Expected current timezone behavior: `Europe/Warsaw`, with `CEST` or `CET` depending on daylight saving time.
- Network time server: `time.apple.com`
- For this stationary homelab Mac, do not rely on automatic timezone/location detection.

## Remote access

- SSH is expected to work at the Mac mini LAN IP `192.168.10.13`.
- Screen Sharing is expected to work.
- AdGuard Home web UI:
  - LAN URL: `http://192.168.10.13:3002`
- MeTube web UI:
  - LAN URL: `http://192.168.10.13:8091`
- UpSnap web UI:
  - LAN URL: `http://192.168.10.13:8090`
  - Bound exclusively to `192.168.10.13:8090`; not available on localhost, 0.0.0.0, or Tailscale

## Login and startup behavior

- Auto-login is enabled and was verified after reboot.
- Reopen previous apps/windows after restart or login is disabled.
- Intended startup behavior:
  - only deliberate Login Items and LaunchAgents should start automatically
  - previously open GUI apps/windows should not be restored automatically after login
- LaunchAgents note:
  - restored apps/windows are unwanted GUI session state from the prior login session
  - LaunchAgents are intentional background services or helpers
  - LaunchAgents are not the same as restored GUI apps/windows

## Power and sleep settings

Verified operator state:

- `sleep 0` (sleep disabled)
- `disksleep 0` (disk sleep disabled)
- `autorestart 1` (auto-restart after power loss enabled)
- `tcpkeepalive 1` (TCP keepalive enabled)
- `womp 1`
- Intentional LaunchAgents still start normally.
- The Cudy router sends a Wake-on-LAN packet to the Mac mini every day at `04:00`.

## Tailscale

- Tailscale app is installed.
- Mac mini is configured as an exit node and subnet router.
- Advertised subnet route: `192.168.10.0/24`
- LAN services should normally use LAN addresses, not Tailscale addresses.

## AdGuard Home

- Installed natively on macOS, not in Docker.
- Current version: `v0.107.78`.
- DNS endpoint: `192.168.10.13:53`
- Web UI: `http://192.168.10.13:3002`
- DHCP: disabled
- Operational role:
  - secondary DNS server for the home network
  - primary DNS remains the Fedora miniPC AdGuard at `192.168.10.12`
- Configuration policy:
  - both AdGuard configurations should remain mirrored because clients may use either server

## MeTube

- Status: installed natively and intended to remain native.
- Installed release: stable tag `2026.07.24` (commit `50250f83743ad21c584ee81b94fd42c4d0743a4b`). Backend: Python `3.13.14`, `yt-dlp 2026.07.04`, `ffmpeg 8.1.2`. UI build toolchain: Node `v22.23.1`, `pnpm@11.5.2`.
- LaunchAgent label: `com.patrykmac.metube`
- LaunchAgent plist path:
  - `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.metube.plist`
- App path:
  - `/Users/patrykmac/homelab/metube/app`
- Python venv path:
  - `/Users/patrykmac/homelab/metube/venv`
- State path:
  - `/Users/patrykmac/homelab/metube/state`
- Launch wrapper path:
  - `/Users/patrykmac/homelab/metube/start-metube.sh`
- iCloud downloads path:
  - `/Users/patrykmac/Library/Mobile Documents/com~apple~CloudDocs/DRIVE/MeTube`
- ffmpeg path:
  - `/opt/homebrew/bin/ffmpeg`
- yt-dlp custom options file path:
  - `/Users/patrykmac/homelab/metube/ytdl-options.json` (referenced by `YTDL_OPTIONS_FILE`; merged over the plist's `YTDL_OPTIONS`; MeTube watches this file and reloads it on change)
- H.264/AAC re-encode safety-net:
  - Script path: `/Users/patrykmac/homelab/metube/scripts/ensure-h264.sh`, mode `700`, invoked as a yt-dlp `Exec` postprocessor (`when: after_move`) once per download.
  - Behavior: checks both video and audio codec via `ffprobe`. Video `!= h264` → full re-encode (`libx264 veryfast crf 20 yuv420p` + `aac 160k`). Video `== h264` but audio `!= aac` → audio-only re-encode (`-c:v copy -c:a aac -b:a 160k -movflags +faststart`; video stream is never touched or re-compressed). Both already `h264`/`aac` → no-op. Uses an atomic same-directory temp-file-then-`mv` replace.
  - `ytdl-options.json` is mode `600`, owned `patrykmac:staff`, and contains `{"format_sort": ["vcodec:h264", "acodec:aac"], "merge_output_format": "mp4"}` plus the `Exec` postprocessor entry that invokes the script above.
- Runtime note:
  - the wrapper exports the Homebrew PATH so `yt-dlp` can find `ffmpeg`
- Verification note:
  - MeTube works after reboot
  - the first post-reboot start may require macOS/iCloud permission for Python
- Full UI build note:
  - full MeTube application updates require a manual UI build outside the Codex sandbox
  - known working UI build command:

```sh
cd /Users/patrykmac/homelab/metube/app/ui
PATH="/opt/homebrew/opt/node@22/bin:/opt/homebrew/bin:$PATH" \
/opt/homebrew/opt/node@22/bin/npx pnpm@11.5.2 run build
```

## MeTube cleanup

- Scheduled launcher:
  - `/Users/patrykmac/homelab/metube/MeTubeCleanupLauncher.app`
  - bundle identifier: `com.patrykmac.metube-cleanup-launcher`
- Access model:
  - The launcher has a macOS security-scoped bookmark for the existing MeTube iCloud folder only, chosen interactively through the Apple file picker.
  - Bookmark path: `/Users/patrykmac/Library/Application Support/MeTubeCleanupLauncher/metube-folder.bookmark`.
  - No Full Disk Access is granted to `/bin/bash`, Terminal, launchd, or any generic interpreter.
- LaunchAgent label:
  - `com.patrykmac.metube-cleanup`
- LaunchAgent plist path:
  - `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.metube-cleanup.plist`
- Schedule:
  - Wednesday and Sunday around `04:20`
- Rule:
  - deletes regular files older than 6 days from the MeTube iCloud folder
  - removes empty non-hidden subdirectories (iCloud Drive does not preserve manually set directory modification times reliably)
  - never deletes the base folder
- Backup and recovery:
  - The canonical archive includes `services/metube/MeTubeCleanupLauncher.app`, copied from `/Users/patrykmac/homelab/metube/MeTubeCleanupLauncher.app` with ACLs, resource forks, extended attributes, and file flags preserved by the backup tooling. The embedded ad-hoc signature is inspected after non-destructive extraction by the restore verifier.
  - The live bundle has no extended attributes; its embedded code signature remains valid. The archive is a recovery copy of the signed bundle, not a replacement or change to the live launcher.
  - The security-scoped bookmark remains deliberately excluded. It is specific to the `patrykmac` user, macOS TCC authorization, and the interactively selected folder, so an archived copy cannot be assumed valid after a restore or on another installation.
  - After a restore, put the bundle back at the documented live path and restore the existing plist unchanged. Log in as `patrykmac`, run `MeTubeCleanupLauncher --authorize` from `MeTubeCleanupLauncher.app/Contents/MacOS`, select exactly `/Users/patrykmac/Library/Mobile Documents/com~apple~CloudDocs/DRIVE/MeTube` in the macOS picker, then run the same executable with `--diagnostic-access`. Do not invoke it without an argument during validation because that starts cleanup.

## MeTube maintenance

- Status script:
  - `/Users/patrykmac/homelab/metube/maintenance/metube-status.sh`
- Manual update procedure:
  - `/Users/patrykmac/homelab/metube/maintenance/UPDATE-METUBE.md`
- Policy:
  - `yt-dlp` updates should be manual and tested
  - full MeTube application updates require a manual UI build outside the Codex sandbox

## External SSD

- Mounted as `/Volumes/SSD-MINI`
- Filesystem: `APFS`
- Not used for MeTube downloads.
- Stores the local homelab service/configuration backup archives at `/Volumes/SSD-MINI/Homelab-Backups`; it is not used by Time Machine.

## Fedora dashboard (Homarr)

- Fedora Homepage has been retired. Homarr is now the Fedora miniPC dashboard.
- Homarr includes only these Mac mini links:
  - Uptime Kuma
  - AdGuard Home
  - MeTube
- Homarr may also use the Mac mini's LAN-only Glances API as a data source for CPU/RAM metrics.
- No Mac mini SSH, router, NAS, Homepage, Homarr, or Docker component is installed locally on this Mac mini; the dashboard entries are remote links only.

## LAN documentation viewer (decommissioned)

- Status: stopped and persistently disabled on 2026-07-26. The former native LAN viewer is decommissioned in favor of the public GitHub mirror: `patryk-homelab/homelab-docs-public`.
- The service has no active listener and must not be re-added as a Fedora Homarr tile.
- Retained for a future deliberate reactivation: implementation `/Users/patrykmac/homelab/homelab-docs/server.py`; LaunchAgent `com.patrykmac.homelab-docs` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-docs.plist`; logs `/Users/patrykmac/homelab/homelab-docs/logs/launchd.stdout.log` and `/Users/patrykmac/homelab/homelab-docs/logs/launchd.stderr.log`.
- The plist, application directory, server implementation, and logs remain on disk unchanged; only the loaded job was stopped and its user-domain LaunchAgent was disabled.
- Backup coverage continues to include `server.py`, this document, and the retained LaunchAgent. Transient logs, bytecode cache, PID files, sockets, and temporary files remain excluded.

## Glances

- Status: installed natively with Homebrew; Docker Desktop and other container runtimes are not used.
- Purpose: expose Mac mini CPU and RAM metrics to Homarr on the Fedora miniPC.
- Homebrew formula/runtime version: Glances `4.5.5_1`.
- Binary: `/opt/homebrew/bin/glances`
- Bind/listen policy:
  - LAN address only: `192.168.10.13`
  - TCP port: `61208`
  - Intended only for the trusted LAN `192.168.10.0/24`
- API endpoint for Homarr:
  - `http://192.168.10.13:61208/api/4/quicklook`
- Additional validated endpoints:
  - `http://192.168.10.13:61208/api/4/cpu`
  - `http://192.168.10.13:61208/api/4/mem`
- Runtime model:
  - user LaunchAgent, runs as `patrykmac`
  - starts automatically after login
  - `KeepAlive` enabled
  - API only (`--disable-webui`) to keep the service lightweight
  - uses `--disable-config-exec` and `--disable-check-update` for safer unattended service behavior
- LaunchAgent label: `com.patrykmac.homelab.glances`
- Canonical LaunchAgent plist path:
  - `/Users/patrykmac/homelab/glances/com.patrykmac.homelab.glances.plist`
- LaunchAgents link path:
  - `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab.glances.plist`
- Service paths:
  - wrapper: `/Users/patrykmac/homelab/glances/start-glances.sh`
  - logs: `/Users/patrykmac/homelab/glances/logs/glances.stdout.log`
  - logs: `/Users/patrykmac/homelab/glances/logs/glances.stderr.log`
- Exact service command:

```sh
/opt/homebrew/bin/glances -w --disable-webui -B 192.168.10.13 -p 61208 --disable-config-exec --disable-check-update
```

- Validation commands:

```sh
plutil -lint /Users/patrykmac/homelab/glances/com.patrykmac.homelab.glances.plist
launchctl print gui/$(id -u)/com.patrykmac.homelab.glances
lsof -nP -iTCP:61208 -sTCP:LISTEN
curl http://192.168.10.13:61208/api/4/quicklook
curl http://192.168.10.13:61208/api/4/cpu
curl http://192.168.10.13:61208/api/4/mem
launchctl kickstart -k gui/$(id -u)/com.patrykmac.homelab.glances
tail -n 100 /Users/patrykmac/homelab/glances/logs/glances.stderr.log
```

- Removal / rollback:

```sh
launchctl bootout gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab.glances.plist
rm /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab.glances.plist
brew uninstall glances
rm -rf /Users/patrykmac/homelab/glances
```

- Resource/interaction note:
  - lightweight native monitoring only; no changes were made to Ollama, MeTube, AdGuard Home, Uptime Kuma, the Discord bot, or the existing maintenance LaunchAgents

## UpSnap

- Status: installed natively on macOS, not in Docker.
- Purpose: LAN Wake-on-LAN management — power on and monitor LAN devices (primarily the Fedora miniPC) from a browser UI.
- Version: `5.4.4` (official GitHub release asset `UpSnap_5.4.4_darwin_arm64.zip`).
- Release SHA-256 (verified at install time): `b621df649044aecad9912447a75ba7fbd0054b317d348fee9dddd4fbf0982767`.
- Binary origin: downloaded from the official GitHub release, SHA-256 verified against `checksums.txt`, quarantine-cleared with `xattr -c`, and ad-hoc signed with `codesign --force --sign -`. No privileged or developer identity is required.
- Web UI: `http://192.168.10.13:8090`
- Bind policy:
  - Listens **only** on `192.168.10.13:8090` (LAN IP, explicit `--http=192.168.10.13:8090` flag).
  - Deliberately **not** bound to `127.0.0.1`, `0.0.0.0`, or the Tailscale address.
  - `http://127.0.0.1:8090` is intentionally unreachable; confirmed by connection-refused curl at install.
- Install directory: `/Users/patrykmac/homelab/upsnap/`
- Binary path: `/Users/patrykmac/homelab/upsnap/upsnap`
- Wrapper path: `/Users/patrykmac/homelab/upsnap/start-upsnap.sh`
- Canonical LaunchAgent plist: `/Users/patrykmac/homelab/upsnap/com.patrykmac.upsnap.plist`
- LaunchAgents symlink: `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.upsnap.plist` → canonical plist (mirrors the Glances pattern)
- Data directory: `/Users/patrykmac/homelab/upsnap/pb_data` (persistent PocketBase state)
- Logs directory: `/Users/patrykmac/homelab/upsnap/logs` (transient; excluded from backup)
- LaunchAgent label: `com.patrykmac.upsnap`
- Runtime model: user LaunchAgent with `RunAtLoad` and `KeepAlive`; starts automatically after login and restarts on exit.
- Wrapper environment and serve flag:

```sh
#!/bin/sh
cd /Users/patrykmac/homelab/upsnap || exit 1
export UPSNAP_WEBSITE_TITLE="UpSnap Mac mini"
export UPSNAP_SCAN_RANGE="192.168.10.0/24"
export UPSNAP_PING_PRIVILEGED="false"
exec /Users/patrykmac/homelab/upsnap/upsnap serve --http=192.168.10.13:8090
```

- `UPSNAP_HTTP_LISTEN` is deliberately **not** set; the `--http` flag is used instead so CLI subcommands remain functional.
- Privileged ping is disabled (`UPSNAP_PING_PRIVILEGED=false`); unprivileged ICMP is used.
- Expected RAM/disk impact: lightweight; the binary is a self-contained Go executable. The only persistent state is `pb_data` (small PocketBase SQLite store). No impact on local LLM capacity.
- Administrator account: **not yet created**. The initial superuser account must be created manually by the user through the browser UI at `http://192.168.10.13:8090/_/`.
- Fedora device: **not yet added**. The Fedora miniPC (`192.168.10.12`, MAC `52:54:00:xx:xx:xx`) will be added manually through the UI after account creation.

```sh
# Status, restart, logs, port verification, HTTP check, disable, and re-enable only UpSnap
launchctl print gui/$(id -u)/com.patrykmac.upsnap
launchctl kickstart -k gui/$(id -u)/com.patrykmac.upsnap
tail -n 100 /Users/patrykmac/homelab/upsnap/logs/launchd.stderr.log
tail -n 100 /Users/patrykmac/homelab/upsnap/logs/launchd.stdout.log
lsof -nP -iTCP:8090 -sTCP:LISTEN
curl -fsS -o /dev/null -w '%{http_code}\n' http://192.168.10.13:8090/
launchctl bootout gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.upsnap.plist
launchctl bootstrap gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.upsnap.plist
```

## Homelab status agent

- Purpose: lightweight, native, read-only Mac mini status endpoint for the Fedora n8n aggregator and a future phone widget. It exposes only a fixed, approved set of Mac mini metrics; it never returns process lists, usernames, network addresses, command lines, serial numbers, UUIDs, file paths, tokens, logs, Discord data, service inventories, SMART details, Tailscale details, MeTube queue data, or backup contents.
- URL: `http://192.168.10.13:3005/status` (and `http://192.168.10.13:3005/health`).
- Bind and source-address policy: binds only to the Mac mini LAN address `192.168.10.13:3005`; not bound to `0.0.0.0`, `127.0.0.1`-only, a Tailscale address, or an IPv6 wildcard. Every request is checked against a fixed source-address allowlist (`192.168.10.12` Fedora miniPC, `192.168.10.13` Mac mini self-check); any other source address receives `403`.
- Routes: `GET`/`HEAD` `/health` and `GET`/`HEAD` `/status` only. Any other path returns `404`; any other method returns `405`.
- Schema (`schema_version: 1`): `host`, `timestamp` (ISO-8601, `Europe/Warsaw`), `cpu.percent`, `ram.percent`/`used_bytes`/`total_bytes`, `storage.internal.{available,free_bytes,total_bytes}`, `storage.ssd_mini.{available,free_bytes,total_bytes}`, `uptime_seconds`, `ollama.{reachable,loaded,models}`, `last_backup.{status,archive,created_at,age_seconds,checksum_file_present}`, and a non-sensitive `errors` array. Unknown/unavailable metrics are `null` (never a fabricated zero) with a concise entry appended to `errors`.
- Data sources:
  - CPU/RAM/uptime/internal filesystem: the existing Glances API at `http://192.168.10.13:61208` (`/api/4/cpu`, `/api/4/mem`, `/api/4/uptime`, `/api/4/fs`). The internal filesystem entry is selected by matching `mnt_point == "/"` in the live Glances response, never by array index. Only these four approved fields are extracted; no other Glances data is proxied or exposed.
  - SSD-MINI capacity: read-only `os.statvfs()` against `/Volumes/SSD-MINI`, gated by the same mount/name/UUID identity check used by the existing SSD-MINI monitor (`SSD-MINI`, APFS, UUID `AE38A773-CCDE-42A2-9127-2C055305D944`). If identity validation fails or the volume is absent, the endpoint reports `"available": false` with `null` capacity fields; it never mounts, repairs, ejects, or modifies the disk, and never sends a Pushover alert (the existing SSD-MINI availability monitor remains the sole authority for that).
  - Ollama: the existing local-only `GET http://127.0.0.1:11434/api/ps`. Reports `reachable`/`loaded`/`models` (names only); never calls a generation endpoint, never loads/unloads a model, and never alters keep-alive. The Ollama API itself remains localhost-only and is not exposed to the LAN by this service.
  - Last backup: lightweight inspection of `/Volumes/SSD-MINI/Homelab-Backups` for the newest `macmini-homelab-YYYY-MM-DD_HHMMSS.tar.gz` with a matching `.sha256` file. Validation only confirms both files exist and that the checksum file matches the expected `shasum -a 256` line format; it never hashes or opens the archive, never triggers a backup, and never contacts or mounts the NAS. Reports a `status` of `ok`, `missing`, or `unavailable` (when SSD-MINI itself is unavailable), the archive basename only, its creation timestamp/age, and whether a checksum file is present — never the checksum value itself.
- Timeout/failure behavior: short timeouts on all upstream calls (Glances ~2s, Ollama ~1.5s); one failed data source never blocks the others. Responses are always freshly computed (no server-side caching) with `Cache-Control: no-store`, `Content-Type: application/json; charset=utf-8`, and restrictive headers (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Content-Security-Policy`).
- Implementation: existing Homebrew Python 3 standard library only (no Flask/FastAPI/Node/Docker/new runtime).
  - Server: `/Users/patrykmac/homelab/status-agent/server.py`
  - Wrapper: `/Users/patrykmac/homelab/status-agent/start-status-agent.sh`
  - Canonical LaunchAgent plist: `/Users/patrykmac/homelab/status-agent/com.patrykmac.homelab-status-agent.plist`
  - LaunchAgents symlink: `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-status-agent.plist` → canonical plist (mirrors the Glances/UpSnap convention)
  - Logs (transient, excluded from backup): `/Users/patrykmac/homelab/status-agent/logs/launchd.stdout.log`, `/Users/patrykmac/homelab/status-agent/logs/launchd.stderr.log`
  - LaunchAgent label: `com.patrykmac.homelab-status-agent`; `RunAtLoad` and `KeepAlive` enabled; explicit minimal `PATH` (`/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin`); runs unprivileged as `patrykmac`; no Full Disk Access or new TCC grants; no root privileges.
- Read-only/no-model-load statement: the service performs no writes anywhere, never mounts/repairs/ejects SSD-MINI, and never loads, unloads, or alters the keep-alive of an Ollama model. It remains lightweight so it does not compete with Ollama/local LLM RAM or disk budget.
- Backup coverage: `server.py`, the wrapper, the canonical plist, and the LaunchAgents-copy plist are included in the canonical homelab backup and required by non-restoring restore verification. Transient logs, bytecode cache, PID files, sockets, and temporary files are intentionally excluded.
- Safe validation commands:

```sh
plutil -lint /Users/patrykmac/homelab/status-agent/com.patrykmac.homelab-status-agent.plist
launchctl print gui/$(id -u)/com.patrykmac.homelab-status-agent
lsof -nP -iTCP:3005 -sTCP:LISTEN
curl http://192.168.10.13:3005/health
curl http://192.168.10.13:3005/status
launchctl kickstart -k gui/$(id -u)/com.patrykmac.homelab-status-agent
tail -n 100 /Users/patrykmac/homelab/status-agent/logs/launchd.stderr.log
launchctl bootout gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-status-agent.plist
launchctl bootstrap gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-status-agent.plist
```

## WOL/recovery

- Wake-on-LAN target interface:
  - Ethernet interface `en0`
- Ethernet MAC:
  - `14:98:77:6b:d2:f9`
- Wake target IP:
  - `192.168.10.13`
- Router recovery behavior:
  - the Cudy router sends a WOL packet to the Mac mini every day at `04:00`
- Manual WOL can be sent to:
  - MAC `14:98:77:6b:d2:f9`
  - broadcast `192.168.10.255`
  - UDP port `9`

## Monitoring plan

- Uptime Kuma on Fedora miniPC remains active.
- Historical Homepage monitor note: the retired `Homepage via reverse proxy` monitor was removed from Fedora Uptime Kuma and must not be re-added.
- The Mac mini Uptime Kuma provides cross-monitoring for the Fedora miniPC.
- It monitors only critical Fedora services, not every application.
- The retired Homepage dashboard is intentionally not monitored.
- Fedora Web Console / Cockpit remains installed and active on the Fedora miniPC, but is intentionally not monitored by Uptime Kuma.
- The removed `miniPC Cockpit` monitor must not be reintroduced unless explicitly requested.

## SSD-MINI availability monitor

- Purpose: a lightweight native user monitor that reports a confirmed `SSD-MINI` loss and sends one recovery notification when the same verified volume returns. It is read-only: it never mounts, remounts, repairs, ejects, reformats, or otherwise modifies the disk.
- Script: `/Users/patrykmac/homelab/monitoring/check-ssd-mini.sh`.
- LaunchAgent: `com.patrykmac.ssd-mini-monitor` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.ssd-mini-monitor.plist`; `RunAtLoad` plus `StartInterval=60` runs it once per minute in the `patrykmac` user domain.
- Fixed production checks: mount path `/Volumes/SSD-MINI`; APFS filesystem; volume name `SSD-MINI`; authoritative APFS UUID `AE38A773-CCDE-42A2-9127-2C055305D944`; readable mount; and readable/metadata-accessible `/Volumes/SSD-MINI/Homelab-Backups`. It does not perform an ordinary-poll write test.
- State files: `/Users/patrykmac/homelab/monitoring/state/ssd-mini.state` and `/Users/patrykmac/homelab/monitoring/state/ssd-mini-failures.state`. Atomic state writes and a user-writable PID lock prevent overlap and retain the state across normal restarts, logouts, and reboots.
- State machine: `unknown` healthy becomes `healthy` without a notification; one failed poll becomes `suspected_missing`; two consecutive failures become `missing` and send one priority-1 failure notification. Further failures do not resend. A healthy result from `suspected_missing` is silent; a healthy result from `missing` sends one priority-0 recovery notification and resets the counter.
- Failure title: `Mac Mini – SSD-MINI missing`; recovery title: `Mac Mini – SSD-MINI restored`. Messages contain only timestamp, mount/name/UUID validation, concise reason, and backup-directory health. A failed delivery is recorded and receives at most one next-poll retry; credentials and response bodies are never logged.
- Credentials: the isolated `/Users/patrykmac/homelab/monitoring/config/pushover.env` is mode `0600`, uses separate `PUSHOVER_APP_TOKEN` and `PUSHOVER_USER_KEY` fields, and is neither embedded in the script/plist nor archived. The non-secret `config/pushover.env.example` is retained for recovery.
- Logs: `/Users/patrykmac/homelab/monitoring/logs/ssd-mini-monitor.log`, rotated to one `.1` predecessor when the active file reaches 2 MiB. LaunchAgent stdout/stderr are in the same log directory.
- Safe test mode: `--dry-run` retains normal health/state logic but records notification events locally; `--simulate-failure <approved_reason>` safely exercises failure paths without touching the disk. Environment overrides exist only for test mount/name/UUID/backup/state/log/notification inputs; production defaults remain fixed.

```sh
# Inspect persisted state and recent monitor events
cat /Users/patrykmac/homelab/monitoring/state/ssd-mini.state
cat /Users/patrykmac/homelab/monitoring/state/ssd-mini-failures.state
tail -n 100 /Users/patrykmac/homelab/monitoring/logs/ssd-mini-monitor.log

# Run one read-only live health check (no notification from a healthy initial state)
/Users/patrykmac/homelab/monitoring/check-ssd-mini.sh

# Disable or re-enable only this monitor
launchctl bootout gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.ssd-mini-monitor.plist
launchctl bootstrap gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.ssd-mini-monitor.plist
```

## Smartmontools physical-health monitoring

- Homebrew formula: `smartmontools 7.5`; native arm64 binary: `/opt/homebrew/bin/smartctl` (the companion daemon is `/opt/homebrew/bin/smartd`). It is installed as a manual diagnostic tool and no `brew services` entry or persistent `smartd` daemon is enabled.
- Internal physical NVMe (`APPLE SSD AP0256Q`, Apple Fabric, resolved via `smartctl --scan-open` as the Apple NVMe IOService path) exposes reliable health data through `smartctl -i -H -A` and is the disk covered by automated monitoring below. A read-only `-x` audit cannot retrieve the separate NVMe Error Information Log (`GetLogPage` system `0x38`, code `745`); this is an Apple-controller API limitation, not a disk failure — the standard health log itself is reliable and is the monitored source.
- SSD-MINI (external USB SSD through the `Realtek RTL9210-VB-CG` bridge) reports `SMART Status: Not Supported` under macOS; neither NVMe nor SATA SMART data is obtainable through the current macOS/enclosure stack across any bridge mode Smartmontools 7.5 supports for this controller, because macOS presents the device as ATA rather than the SCSI transport those modes require. This unsupported result is not a disk failure. SSD-MINI is therefore intentionally not SMART-polled; its separate availability monitor (mount/name/UUID/backup-path checks, above) remains authoritative for SSD-MINI health.
- Automated monitor: `/Users/patrykmac/homelab/monitoring/check-smart-health.sh` resolves the scan-discovered Apple NVMe each run and then verifies its configured model/serial allowlist in `/Users/patrykmac/homelab/monitoring/config/smart-health.env`. Persistent non-secret state is `/Users/patrykmac/homelab/monitoring/state/smart-health.state`; logs are `/Users/patrykmac/homelab/monitoring/logs/smart-health-monitor.log` (bounded to 2 MiB plus `.1`), with separate bounded launchd stdout/stderr paths.
- LaunchAgent: `com.patrykmac.smart-health-monitor` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.smart-health-monitor.plist`, scheduled once daily at `02:30` local time via `StartCalendarInterval`. It does not poll tightly, overlap executions, start SMART self-tests, or alter disk power settings.
- Alert policy: every Pushover notification emitted by `check-smart-health.sh` uses the centralized exact title `Mac Mini - Disk Alert`; this includes synthetic test, critical health/counter, access, temperature, and recovery notifications. It uses the existing approved notifier without copying credentials. It alerts once for newly observed critical-warning/overall-health changes or increased media/error-log counters; access failure requires two consecutive checks. It has no routine all-clear. A single recovery notice is sent only when an access or temperature condition clears, never merely because a permanent media-error counter remains unchanged. `smartctl` return values are interpreted as the installed manual's bitmask (bits 0–2 command/open/SMART-command failure, bit 3 failing status, bits 4–7 ATA health/log conditions); the monitor treats access bits separately from health fields.
- SSD-MINI interaction: this monitor has no external-device check and does not duplicate the existing once-per-minute `check-ssd-mini.sh` mount/name/UUID/backup-directory authority. SSD-MINI absence therefore cannot create duplicate SMART notifications.
- Manual safe audit and validation commands (all read-only; do not add `-t`):

```sh
/opt/homebrew/bin/smartctl --scan-open
/opt/homebrew/bin/smartctl -i -H -A '<scan-open Apple NVMe IOService path>'
/Users/patrykmac/homelab/monitoring/check-smart-health.sh --dry-run
/Users/patrykmac/homelab/monitoring/check-smart-health.sh --test-notification
launchctl print gui/$(id -u)/com.patrykmac.smart-health-monitor
```

- Backup/restore: the script, non-secret allowlist/threshold config and example, and LaunchAgent are included in the canonical service archive and required by non-restoring verification. Runtime state, locks, and logs are intentionally excluded so stale anti-spam/access state is not restored; a restored checker establishes a fresh baseline. Formula reconstruction is also captured by the generated Homebrew metadata. No SMART self-tests are scheduled.

## Uptime Kuma

- Status: installed natively on macOS; Docker Desktop is not installed or used.
- Purpose: cross-monitor the Fedora miniPC and its critical services.
- URL: `http://192.168.10.13:3003`
- Release: Uptime Kuma `2.4.0`.
- Database: SQLite.
- App path: `/Users/patrykmac/homelab/uptime-kuma/app`
- Data path: `/Users/patrykmac/homelab/uptime-kuma/data`
- Logs path: `/Users/patrykmac/homelab/uptime-kuma/logs`
- Wrapper path: `/Users/patrykmac/homelab/uptime-kuma/start-uptime-kuma.sh`
- LaunchAgent label: `com.patrykmac.uptime-kuma`
- LaunchAgent plist path: `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.uptime-kuma.plist`
- Runtime model: user LaunchAgent with `RunAtLoad` and `KeepAlive`; it uses the Homebrew Node installation and listens on port `3003`.
- Pushover notifications are enabled for the active monitors.

### Active cross-monitoring

1. `AdGuard - MiniPC`
   - Type: DNS
   - Resolver: `192.168.10.12`
   - Port: `53`
   - Record type: `A`
2. `MiniPC - Uptime Kuma`
   - Type: HTTP(s)
   - URL: `http://192.168.10.12:3001`
   - Accepted status: `200-299`
3. `MiniPC Ping`
   - Type: Ping
   - Host: `192.168.10.12`

### Monitoring policy

- Pushover is enabled for all three active monitors.
- Homepage is intentionally not monitored; `Homepage via reverse proxy` must not be reintroduced.
- Fedora Web Console / Cockpit remains active on the Fedora miniPC but is intentionally not monitored; `miniPC Cockpit` must not be reintroduced unless explicitly requested.
- The Mac mini Uptime Kuma monitors only critical Fedora services, not every application.

### Manual update policy

- Read-only status script: `/Users/patrykmac/homelab/uptime-kuma/maintenance/uptime-kuma-status.sh`
- Manual update and rollback procedure: `/Users/patrykmac/homelab/uptime-kuma/maintenance/UPDATE-UPTIME-KUMA.md`
- Before every update, create and retain a timestamped backup of the entire Uptime Kuma data directory, wrapper, LaunchAgent plist, and current Git tag/commit.
- Update only to a reviewed, specific stable release tag; never update directly to `master`, nightly, or development branches.
- SQLite/database migrations may be one-way. Do not roll back against a newer migrated database without restoring the matching data backup.

## Daily update report

- Purpose: lightweight, read-only daily availability report for the Mac mini's documented critical software. It never installs, downloads, upgrades, or restarts software.
- Script: `/Users/patrykmac/homelab/update-report/macmini-update-report.sh`.
- LaunchAgent: `com.patrykmac.update-report` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.update-report.plist`, scheduled daily at `10:00` in the Mac's configured `Europe/Warsaw` timezone.
- Checks: macOS via `softwareupdate -l`; Homebrew formulae/casks using existing metadata; MeTube's exact installed stable Git tag against the official latest stable GitHub release (drafts, prereleases, `master`/`main`/`HEAD`, and unreleased commits are ignored); the MeTube venv's `yt-dlp` PyPI release; Uptime Kuma and AdGuard Home stable GitHub releases; UpSnap stable GitHub release (installed version detected via `upsnap --version`, falling back to the documented pinned version `5.4.3`).
- Tailscale is installed through the App Store and remains outside this report: no Tailscale version query or Tailscale text is emitted in Pushover update notifications.
- Pushover title: `Mac Mini - Update Report`. The report notifies only when updates are available or an operational check fails; it sends no all-clear notification. It reuses the backup Pushover helper and its existing private credentials file.
- Logs: `/Users/patrykmac/homelab/update-report/logs/current.log`, with up to three rotated historical logs, plus LaunchAgent stdout/stderr logs in the same directory.
- Manual report without sending Pushover: `/Users/patrykmac/homelab/update-report/macmini-update-report.sh --dry-run`.
- Pushover delivery test only: `/Users/patrykmac/homelab/update-report/macmini-update-report.sh --test-notification`.

## Ollama local LLM runtime

- Status: installed natively with Homebrew; no Docker Desktop, Open WebUI, LM Studio, or additional model manager was installed.
- Homebrew formula/runtime version: Ollama `0.32.3` (native `arm64`).
- Binary: `/opt/homebrew/bin/ollama` (Homebrew target: `/opt/homebrew/opt/ollama/bin/ollama`).
- Startup: the single Homebrew user LaunchAgent `homebrew.mxcl.ollama` at `/Users/patrykmac/Library/LaunchAgents/homebrew.mxcl.ollama.plist` uses `RunAtLoad` and `KeepAlive`. It starts after login without a Terminal window and was verified to restart after a controlled SIGTERM.
- Logs: `/opt/homebrew/var/log/ollama.log` (combined stdout/stderr).
- API security posture: listening only on `127.0.0.1:11434`; a probe to `192.168.10.13:11434` was refused. Do not set `OLLAMA_HOST` to a LAN/Tailscale address or expose port `11434`.
- Primary installed model: `qwen3:14b`, digest `bdbd181c33f2…`, stored in the internal Mac mini SSD model store `/Users/patrykmac/.ollama/models`. It is the selected model for the current Mac mini workload; controlled use with one non-thinking inference at a time and a five-minute keep-alive kept existing services operational.
- Non-thinking mode for the future bot: set the per-request top-level Ollama API field to `"think": false`. This was verified through `/api/generate` and `/api/chat`: every test returned `thinking: null`, no `<think>` output, and streamed chunks had no thinking values. The primary CLI equivalent is `ollama run qwen3:14b --think=false "prompt"`. Do not permanently modify the base model.
- Primary keep-alive policy: use a per-request `"keep_alive":"5m"` for normal Discord activity, one inference at a time, then send `{"model":"qwen3:14b","keep_alive":0}` to `/api/generate` on deliberate bot shutdown or maintenance. Do not preload the model permanently or load it during backup/heavy maintenance when avoidable.

### Model capacity policy

- `qwen3:14b` is the approved primary model for short, non-concurrent Discord use, with existing services remaining operational while it's loaded. Exact capacity-benchmark figures are intentionally not retained in this source-of-truth document.
- The 16 GiB Mac mini is approved for one short, non-concurrent `qwen3:14b` request with a five-minute keep-alive; avoid backups and other heavy maintenance while it is loaded. The external `qwen3:8b` remains emergency fallback only. Do not run the models concurrently or test larger models without a new controlled capacity decision.
- `[SILENCE]` remains an optional model convention that the bot validates before posting; normal operation uses bounded output and an application-side length check.

### Ollama manual commands

```sh
# Service status, lifecycle, and logs
brew services info ollama
brew services start ollama
brew services stop ollama
brew services restart ollama
tail -f /opt/homebrew/var/log/ollama.log

# Models and a local non-thinking test
ollama list
ollama show qwen3:14b --verbose
ollama run qwen3:14b --think=false 'Odpowiedz po polsku jednym zdaniem: test działa.'

# Bounded private local API test (non-streaming)
curl --connect-timeout 2 --max-time 30 http://127.0.0.1:11434/api/generate \
  -d '{"model":"qwen3:14b","prompt":"Odpowiedz po polsku krótko.","think":false,"stream":false,"keep_alive":"5m","options":{"num_predict":120}}'

# Inspect/unload loaded models
ollama ps
curl --connect-timeout 2 --max-time 30 http://127.0.0.1:11434/api/generate \
  -d '{"model":"qwen3:14b","keep_alive":0}'
```

### Isolated external emergency fallback

- Primary Ollama remains the single normal Homebrew LaunchAgent. It listens only on `127.0.0.1:11434`, uses the internal store `/Users/patrykmac/.ollama/models`, and contains only `qwen3:14b`.
- The emergency fallback model is `qwen3:8b`, isolated at `/Volumes/SSD-MINI/Ollama-Fallback/models`. Its dedicated server is on demand only, listens only on `127.0.0.1:11435`, and is not a LaunchAgent or other always-running service.
- The fallback uses the same `/opt/homebrew/bin/ollama` binary with `OLLAMA_HOST=127.0.0.1:11435`, `OLLAMA_MODELS=/Volumes/SSD-MINI/Ollama-Fallback/models`, and `OLLAMA_KEEP_ALIVE=0`. This separate-store behavior was proven with an empty external `/api/tags` result before pulling 8B; the external API then listed only 8B while the internal API listed only 14B.
- Never move, symlink, copy, or delete individual Ollama blobs or manifests. Model installation/removal is performed only with supported `ollama pull` and `ollama rm` commands against the appropriate server.
- External fallback controls (all use absolute paths):

```sh
/Users/patrykmac/homelab/ollama-fallback/status-fallback.sh
/Users/patrykmac/homelab/ollama-fallback/start-fallback.sh
/Users/patrykmac/homelab/ollama-fallback/test-fallback.sh
/Users/patrykmac/homelab/ollama-fallback/stop-fallback.sh
```

- `start-fallback.sh` checks that `SSD-MINI` is mounted APFS with UUID `AE38A773-CCDE-42A2-9127-2C055305D944`, writable, and at the expected mount path; it refuses to start if primary Ollama has a loaded model or if port `11435` belongs to an unexpected process. `stop-fallback.sh` unloads 8B and terminates only the PID recorded by the controlled starter. `test-fallback.sh` performs one bounded `"think":false` request then stops it. No fallback service starts automatically.
- Never run 14B and 8B inference concurrently. The helper blocks fallback start while primary is loaded; the future Discord bot must unload/finish primary work before it starts fallback work.
- If `SSD-MINI` is absent, the primary 14B service continues normally. The helpers fail clearly and do not create a local substitute directory or download 8B to internal storage. Future Discord-side fallback design only: unload/finish primary work, verify SSD-MINI, start fallback, use `127.0.0.1:11435`, and log fallback activation. No Discord integration was created here.
- Storage: the external fallback store (`qwen3:8b`) consumes approximately 4.87 GiB; the primary internal store (`qwen3:14b`) uses approximately 8.64 GiB on disk. Both Ollama APIs are localhost-only; LAN and Tailscale probes to both ports are blocked.

## Klefedroniarz Discord bot

- Service name: `Klefedroniarz`, implemented natively in Python at `/Users/patrykmac/homelab/discord-bot`; Docker is not used.
- Discord scope is hard restricted in code and configuration to channel ID `392016630772662277`. It ignores all other channels, threads, DMs, servers, system/empty messages, its own messages, and messages from other bots.
- Token storage: `/Users/patrykmac/homelab/discord-bot/secrets.env`, mode `0600`. It is read only at runtime, is not copied into source/plists/logs/arguments, and is never printed. The file is intentionally excluded by `.gitignore`.
- Primary generation: native Homebrew Ollama `qwen3:14b` on the internal SSD at `http://127.0.0.1:11434`, localhost-only. It uses one serialized inference at a time, `"think":false`, `"keep_alive":"5m"`, `num_predict=90`, `temperature=0.65`, `top_p=0.82`, `repeat_penalty=1.12`, and a bounded HTTP timeout. Visible `<think>` content is stripped defensively.
- Emergency fallback: external `qwen3:8b` at `http://127.0.0.1:11435`, using the existing guarded SSD-MINI helpers. It is used only for primary API/model/timeout/generation failures, never for `[SILENCE]`. Primary is explicitly unloaded before fallback start; fallback is explicitly unloaded and stopped after its one request. The two models must never be loaded concurrently.
- Trigger behavior: a Discord mention, direct reply to the bot, or bounded case-insensitive alias (`klefedroniarz`, `klefedoniarz`, `klefedron`) always queues a reply attempt. Ordinary messages pass a 65% spontaneous gate and, after a successful spontaneous post, have a three-minute cooldown. Direct triggers bypass the cooldown. There is no daily reply limit; exact `[SILENCE]` output suppresses posting.
- Context and memory: direct triggers use up to 11 recent relevant human messages and spontaneous triggers up to 7 (hard ceiling 25 only for a future explicit multi-message case). The trigger appears once, separately labelled, with reply previews and bot/system/empty messages excluded. Local SQLite at `/Users/patrykmac/homelab/discord-bot/data/memory.db` is mode `0600` in a `0700` directory and stores only selected concise user memories, channel memes, and deletion/audit metadata; it never stores raw history, DMs, other channels, credentials, contact/financial/location/medical/sexual/political/religious data, protected traits, or inferred allegations. Retrieval is strong normalized-keyword based, fallible, and capped at two user plus two channel memories, with three total unless a future exceptional direct-match rule is added. The frequency-based channel-style profile is disabled pending a semantic implementation; its table remains for compatibility and is not injected. Per-user memories cap at 30; channel memories cap at 150; unused low-value memories age out after 90 days (relevance decays after 60 days). `/memory`, `/forgetme`, `/forget <memory_id>`, and `/memory-status` are available only in the configured channel and never expose other users' memories. Maintenance: `/Users/patrykmac/homelab/discord-bot/memory-maintenance.sh`. The existing homelab backup includes the small database and bot configuration but excludes `secrets.env`, logs, runtime state, and the venv. Primary generation uses `temperature=0.65`, `top_p=0.82`, `repeat_penalty=1.12`, and up to 90 tokens. A serialized validator can return `OK`, `REWRITE`, or `SILENCE`; one rewrite uses temperature `0.45`, three context messages, and no memories.
- Personality/language: the editable prompt at `/Users/patrykmac/homelab/discord-bot/system-prompt.txt` makes normal Klefedroniarz replies mainly Polish, concise, deliberately much more malicious, mocking, contemptuous, and ridicule-focused. It uses an original crude absurd-comedy register drawing broadly from adult-animation energy, block-estate and rural parody, low-budget sci-fi, workshop failures, and disco-polo/wedding-party conventions; references are used only when they sharpen the immediate joke. Direct relevance and coherence remain mandatory, with no assistant-like filler, username prefixes, unrelated topic changes, lecturing, moderation, or joke explanations. The prompt prohibits direct quotation, lyrics, recognizable catchphrases, close paraphrase, and impersonation of specific copyrighted characters or productions; it requires original metaphors and insults. `/a` remains neutral, local-only, and unaffected by this personality prompt.
- LaunchAgent: `com.patrykmac.klefedroniarz` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.klefedroniarz.plist`. It uses the dedicated virtual environment interpreter through `/Users/patrykmac/homelab/discord-bot/run-bot.sh`, has `RunAtLoad` and `KeepAlive`, writes dedicated launchd logs, and relies on an application runtime lock to prevent duplicate instances.
- Logs: rotating structured application log `/Users/patrykmac/homelab/discord-bot/logs/bot.log`; launchd stdout/stderr logs are in the same directory. Logs record connection, channel, trigger type, primary/fallback selection, latency, silence/cooldown decisions, and errors, but not raw prompts, token, or full conversation history.
- Security: both Ollama APIs remain localhost-only; Discord sends use `allowed_mentions=none`, so accidental `@everyone`/`@here` or user pings are not emitted. No Discord roles/permissions or additional channel access were changed.
- Assistant command `/a`: `/a q:<pytanie>` is a deferred guild-scoped slash command whose short `q` field is Discord's required command input. `/s` is likewise guild-scoped, with no options. Both are explicitly synchronized for the one resolved guild and accepted only in channel `392016630772662277` (not DMs, threads, other channels, or other servers); out-of-scope use is rejected ephemerally. The bot installation requires Discord's `bot` and `applications.commands` OAuth2 scopes. Plain text beginning with `/a` or `/s` is not a command and is ignored by normal personality triggering. `/a` is intentionally local-only: it uses `/Users/patrykmac/homelab/discord-bot/assistant-prompt.txt`, not `system-prompt.txt`, and the serialized localhost `qwen3:14b` queue with `think:false`, temperature `0.25`, bounded output, and `keep_alive:5m`. It makes no external search/API/page request and never provides live verification. For current information it says that local knowledge may be outdated. It is separated from personality, spontaneous behavior, coherence validation, and all memory retrieval/storage. The guarded 8B fallback remains sequential emergency behavior only.
- Copyright behavior for `/a`: full lyrics, books, and articles are not reproduced. The assistant cannot search for them; it may give a short summary or very short excerpt if known and suggests an official or licensed source without inventing a URL.
- Status command `/s`: a compact ephemeral Discord embed, also restricted to the configured channel, performs no LLM inference and never loads a model. It reports bot/Gateway/channel state, uptime, queue, active generation, last successful reply timing; primary Ollama API/model-loaded state, configured keep-alive, measurable expiry and memory; fallback 8B/SSD-MINI/helper state; and physical RAM, memory-pressure/swap, bot and Ollama RSS, plus AdGuard Home, Uptime Kuma, MeTube, and Tailscale state. It also states `/a` mode `local only` and Internet search `disabled`; it excludes prompts, history, paths beyond the documented service helpers, and all secrets.
- Context/privacy behavior: raw Discord display names are no longer passed to the model as dialogue prefixes. Prompt context uses only stable labels (`USER_CURRENT`, `USER_OTHER_n`), while user IDs and memory isolation remain unchanged. The system prompt and final sanitizer prevent a visible reply from beginning with a known display name, neutral context label, or generic `User:`/`Assistant:` dialogue prefix; names can still occur naturally later when relevant.

### Manual commands

```sh
# Local tests only: no Discord connection or messages
/Users/patrykmac/homelab/discord-bot/test-bot.sh
/Users/patrykmac/homelab/discord-bot/test-ollama.sh

# Bot and LaunchAgent status
/Users/patrykmac/homelab/discord-bot/status-bot.sh
launchctl print gui/$(id -u)/com.patrykmac.klefedroniarz

# Normal service restart and logs
launchctl kickstart -k gui/$(id -u)/com.patrykmac.klefedroniarz
tail -f /Users/patrykmac/homelab/discord-bot/logs/bot.log

# `/a` is permanently local-only; it has no external search configuration.
# `/s` is available inside the configured Discord channel and does not load a model.

# Manual foreground-style helper controls (use only when LaunchAgent is not loaded)
/Users/patrykmac/homelab/discord-bot/start-bot.sh
/Users/patrykmac/homelab/discord-bot/stop-bot.sh
```

### Troubleshooting and remaining Discord-side verification

- If the bot appears offline, do not rely on launchd alone: check `bot.log` for `discord_gateway_connected`, authenticated `discord_ready` (user, guild count, and channel resolution), and periodic `discord_gateway_health` entries; also review `launchd.stderr.log` for login, intent, gateway-close, DNS/TLS, or exception errors. Confirm the secrets file remains mode `0600` without opening or printing it.
- If it connects but does not see message text, enable the Discord application's Message Content privileged intent and confirm the bot has View Channel, Read Message History, Send Messages, and Send Messages in Threads only as required for the one configured channel. Do not broaden channel scope.
- Live trigger behavior needs a human-controlled minimal test in channel `392016630772662277`: one direct mention, one alias, one reply to a bot message, and ordinary messages sufficient to observe the gate/cooldown. Do not create test spam. `[SILENCE]` must not be posted.
- If primary Ollama fails, inspect the primary service first. The bot will only then use the external SSD fallback sequence; if SSD-MINI is absent or fallback cannot start, it logs the failure and sends no fabricated reply.

## Homelab-repo Git snapshot (Forgejo)

- Purpose: a local Git repository that snapshots an explicit, hand-maintained allowlist of this host's config/scripts/LaunchAgents into version-controlled history, mirrored near-real-time to a self-hosted Forgejo remote. This is a versioning/history layer, separate from and in addition to the tar.gz service backup documented below.
- Local repository: `/Users/patrykmac/homelab-repo`, a normal Git repo that the collector only ever writes into (one-way, append-only snapshot target; never read from or restored into by anything else).
- Collector: `/Users/patrykmac/homelab/backup/collect-homelab-repo-macmini.sh` (mode `700`). Reads the explicit `SRC<TAB>DST` allowlist at `/Users/patrykmac/homelab/backup/config/homelab-repo-macmini-allowlist.txt` (mode `600`, 25 entries, no wildcards, no whole-directory copies), verifies every source exists, runs the secret-scan gate library `/Users/patrykmac/homelab/backup/lib/secret-scan-homelab-repo.sh` (mode `700`) over the live sources before anything is copied — the entire run aborts with nothing copied or committed if any potential secret literal is found — then copies the allowlisted files into the repo, `git add -A`, and commits only if something actually changed. An `mkdir`-based lock file serializes concurrent runs; a second concurrent invocation fails fast with a clear log message instead of racing.
- Auto-push: after a successful commit (something was actually staged), the collector also runs `git push origin main`. The pre-existing "nothing changed" branch exits before that point, so an idle run makes zero network calls.
- Forgejo remote: a private, dedicated repository `patryk/homelab-repo-mm` on the self-hosted Forgejo instance already running on the Fedora miniPC (`http://192.168.10.12:3300` web UI; port `2222` for Forgejo's own Git-over-SSH server, separate from that host's regular sshd on port `22`). This is a new, separate repository from the Fedora miniPC's own unrelated `homelab-repo`, deliberately kept apart to avoid cross-host push conflicts. It is used only as a plain git+ssh remote; no Fedora miniPC configuration was touched.
- Deploy key: a dedicated ed25519 keypair, `~/.ssh/forgejo-homelab-repo-mm-deploy` (private key mode `600`), generated specifically for this push and not reused elsewhere. Registered as a repository-scoped Deploy Key (not an account-level key) on `patryk/homelab-repo-mm` with Write Access, so it can only push to this one repository.
- SSH configuration: a dedicated `Host forgejo-homelab-repo-mm` alias in `~/.ssh/config` (`HostName 192.168.10.12`, `Port 2222`, `User git`, `IdentityFile` pointing at the deploy key, `IdentitiesOnly yes`). No other SSH configuration was changed. `origin` in `/Users/patrykmac/homelab-repo` is `forgejo-homelab-repo-mm:patryk/homelab-repo-mm.git`.
- Near-real-time trigger: the `com.patrykmac.homelab-repo-watch` user LaunchAgent (`/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-repo-watch.plist`) uses launchd's native `WatchPaths` key — the macOS equivalent of a systemd `.path` unit — to run the collector whenever any allowlisted source file changes. This is near-real-time, not literally instant: it is driven by macOS FSEvents/kqueue with no hard latency guarantee.
- The `WatchPaths` array is generated, not hand-typed, by `/Users/patrykmac/homelab/backup/generate-watchpaths-plist.sh` (mode `700`), which reads the source (`SRC`) column of the same allowlist the collector itself reads, so the watched-path list and the collected-file list cannot drift apart. Re-run this script (then reload the LaunchAgent) after any allowlist edit. The generated plist currently contains exactly the 25 allowlist source paths.
- Daily fallback: the same plist also carries a `StartCalendarInterval` entry at `03:20` (immediately after the existing `03:15` full backup), so a change is still picked up even if a `WatchPaths` event happens while the Mac is asleep or before the agent is loaded after a reboot.
- No other existing LaunchAgent, service, or the Fedora miniPC was touched by this feature.

```sh
# Status of the watch/fallback LaunchAgent
launchctl print gui/$(id -u)/com.patrykmac.homelab-repo-watch

# Manual collector run (normally only triggered by WatchPaths or the 03:20 fallback)
/Users/patrykmac/homelab/backup/collect-homelab-repo-macmini.sh

# Regenerate the WatchPaths plist after editing the allowlist, then reload
/Users/patrykmac/homelab/backup/generate-watchpaths-plist.sh
launchctl bootout gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-repo-watch.plist
launchctl bootstrap gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-repo-watch.plist

# History and remote state
git -C /Users/patrykmac/homelab-repo log --oneline
git -C /Users/patrykmac/homelab-repo ls-remote origin main
```

## Homelab-docs-public GitHub push

- Purpose: a one-way, near-real-time push of only this document (`HOMELAB-MACMINI.md`) to the same public GitHub repository the Fedora miniPC already pushes its `HOMELAB.md` to, so both source-of-truth files live at one public location for GPT/Claude to fetch via GitHub's raw content API. This is separate from and unrelated to the homelab-repo Git snapshot (Forgejo) documented above; it never touches `/Users/patrykmac/homelab-repo` or that pipeline's deploy key, collector, or LaunchAgent.
- GitHub repository: `patryk-homelab/homelab-docs-public` (public). The Fedora miniPC pushes `HOMELAB.md` there independently; this mechanism pushes only `HOMELAB-MACMINI.md` and never reads or modifies `HOMELAB.md`.
- Local working copy: `/Users/patrykmac/homelab-docs-public-github`, a normal Git repo that the push script only ever writes into (one-way, append-only; never read from or restored into by anything else).
- Push script: `/Users/patrykmac/homelab/backup/push-homelab-docs-public-github-macmini.sh` (mode `700`). Reads the single watched source path from `/Users/patrykmac/homelab/backup/config/homelab-docs-public-github-watchpath.txt` (mode `600`), verifies the source exists, runs the existing secret-scan gate library `/Users/patrykmac/homelab/backup/lib/secret-scan-homelab-repo.sh` over the live source before anything is copied — the run aborts with nothing copied or committed if any potential secret literal is found — then copies `HOMELAB-MACMINI.md` into the working copy, `git add`, and commits only if something actually changed. An `mkdir`-based lock file serializes concurrent runs.
- Auto-push: after a successful commit, the script runs `git push origin main`. The "nothing changed" branch exits before that point, so an idle run makes zero network calls. Commit messages are `Snapshot <ISO8601 UTC timestamp>`.
- Push is racy-safe: `push_with_retry` fetches and rebases onto `origin/main` before every push, with a bounded 3-attempt retry, to handle the two-independent-writer race with the Fedora side without ever force-pushing.
- Deploy key: a dedicated ed25519 keypair, `~/.ssh/github-homelab-docs-public-mm-deploy` (private key mode `600`), generated specifically for this push and not reused elsewhere (including the Forgejo deploy key). Registered as a second repository-scoped Deploy Key (not an account-level key) on `homelab-docs-public` with Write Access, alongside Fedora's own deploy key, so it can only push to this one repository.
- SSH configuration: a dedicated `Host github-homelab-docs-public-mm` alias in `~/.ssh/config` (`HostName github.com`, `User git`, `IdentityFile` pointing at the deploy key, `IdentitiesOnly yes`). No other SSH configuration was changed. `origin` in `/Users/patrykmac/homelab-docs-public-github` is `github-homelab-docs-public-mm:patryk-homelab/homelab-docs-public.git`.
- Near-real-time trigger: the `com.patrykmac.homelab-docs-public-github-watch` user LaunchAgent (`/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-docs-public-github-watch.plist`) uses launchd's native `WatchPaths` key to run the push script whenever `HOMELAB-MACMINI.md` changes. This is near-real-time, not literally instant: it is driven by macOS FSEvents/kqueue with no hard latency guarantee.
- The `WatchPaths` array is generated, not hand-typed, by `/Users/patrykmac/homelab/backup/generate-watchpaths-plist-homelab-docs-public-github.sh` (mode `700`), which reads the same single-path config file the push script itself reads, so the watched path and the pushed file can never drift apart. Re-run this script (then reload the LaunchAgent) after any change to the config.
- Daily fallback: the same plist also carries a `StartCalendarInterval` entry at `03:25` (a few minutes after the existing homelab-repo-watch `03:20` fallback, so the two don't collide), so a change is still picked up even if a `WatchPaths` event happens while the Mac is asleep or before the agent is loaded after a reboot.
- No other existing LaunchAgent, service, the homelab-repo/Forgejo pipeline, or the Fedora miniPC was touched by this feature.

```sh
# Status of the watch/fallback LaunchAgent
launchctl print gui/$(id -u)/com.patrykmac.homelab-docs-public-github-watch

# Manual push run (normally only triggered by WatchPaths or the 03:25 fallback)
/Users/patrykmac/homelab/backup/push-homelab-docs-public-github-macmini.sh

# Regenerate the WatchPaths plist after editing the config, then reload
/Users/patrykmac/homelab/backup/generate-watchpaths-plist-homelab-docs-public-github.sh
launchctl bootout gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-docs-public-github-watch.plist
launchctl bootstrap gui/$(id -u) /Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-docs-public-github-watch.plist

# History, remote state, and public fetch check
git -C /Users/patrykmac/homelab-docs-public-github log --oneline
git -C /Users/patrykmac/homelab-docs-public-github ls-remote origin main
curl https://raw.githubusercontent.com/patryk-homelab/homelab-docs-public/main/HOMELAB-MACMINI.md
```

## iCloud homelab document exchange

- Shared iCloud folder (verified on disk): `/Users/patrykmac/Library/Mobile Documents/com~apple~CloudDocs/Desktop/HOMELAB`. The alternate Finder Desktop path `/Users/patrykmac/Desktop/HOMELAB` is not present on this host.
- Fedora-to-Mac receiver: the Fedora miniPC's dedicated public key is an untrusted, narrowly scoped credential in `/Users/patrykmac/.ssh/authorized_keys`. Its entry is restricted to `command="/Users/patrykmac/homelab/backup/receive-homelab-md.sh",restrict`; the forced-command script ignores `SSH_ORIGINAL_COMMAND`, reads standard input only, and atomically writes exactly `HOMELAB.md` in the shared iCloud folder using a same-directory temporary file and `mv`. It provides no shell, forwarding, or other file access.
- Mac-to-iCloud push: `/Users/patrykmac/homelab/backup/push-homelab-macmini-icloud.sh` (mode `700`) atomically mirrors only the canonical source `/Users/patrykmac/homelab/HOMELAB-MACMINI.md` to `HOMELAB-MACMINI.md` in that folder. It compares content first and exits without writing when unchanged.
- Automatic trigger: user LaunchAgent `com.patrykmac.homelab-macmini-icloud-push` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-macmini-icloud-push.plist` uses `WatchPaths` for the canonical source. It is independent of and additional to the existing GitHub mirror push.
- Recovery/backup coverage: the receiver script, push script, and LaunchAgent are included in the canonical homelab backup. iCloud Drive contents themselves remain excluded from that archive.

```sh
# Manual atomic push and LaunchAgent status
/Users/patrykmac/homelab/backup/push-homelab-macmini-icloud.sh
launchctl print gui/$(id -u)/com.patrykmac.homelab-macmini-icloud-push
```

## Backup

### Time Machine

- Full-system Time Machine backup is configured directly to the Synology NAS.
- Time Machine does not use `SSD-MINI` and remains separate from the homelab service/configuration backup.
- Do not mix service backup archives with the Time Machine share.

### Service backup architecture

- Implementation directory: `/Users/patrykmac/homelab/backup`.
- Main script: `/Users/patrykmac/homelab/backup/backup-macmini-homelab.sh`.
- Non-restoring validation: `/Users/patrykmac/homelab/backup/restore-verify-macmini-homelab.sh`.
- Local destination: `/Volumes/SSD-MINI/Homelab-Backups`.
- Dedicated NAS share: `macmini-backup-ssd`; observed mounted path: `/Volumes/macmini-backup-ssd`, with archives under `Homelab-Backups`.
- LaunchAgent: `com.patrykmac.homelab-backup` at `/Users/patrykmac/Library/LaunchAgents/com.patrykmac.homelab-backup.plist`, scheduled daily at `03:15` local time.
- The LaunchAgent invokes the dedicated app launcher `/Users/patrykmac/homelab/backup/MacMiniHomelabBackupLauncher.app/Contents/MacOS/MacMiniHomelabBackupLauncher` (bundle identifier `com.patrykmac.homelab-backup-launcher`) while preserving the `03:15` schedule and existing stdout/stderr paths.
- The launcher has only macOS removable-volume permission (`kTCCServiceSystemPolicyRemovableVolumes`) for writing homelab archives to SSD-MINI; it does not have Full Disk Access and does not grant access to generic `/bin/bash`, Terminal, or launchd.
- The launcher passed one interactive and one disposable user-LaunchAgent SSD write/delete diagnostic before production use.
- Preflight identifies the expected APFS volume by UUID `AE38A773-CCDE-42A2-9127-2C055305D944`. It distinguishes a physically absent SSD, a detected but unmounted APFS volume (with one bounded mount attempt), an unexpected mount path, a missing backup directory, and a non-writable destination. A device absent from both `diskutil list` and the USB tree is a USB/device-path event, not an ordinary unmounted-volume condition.

### Scope and intentional exclusions

- Included: complete native Uptime Kuma application repository, `node_modules`, wrappers, maintenance documentation, and a consistent `kuma.db` snapshot; complete native MeTube repository, Python venv, UI dependencies, wrappers, cleanup script, minimal state, maintenance documentation, the `scripts/` directory (currently the `ensure-h264.sh` iPhone-playback safety net), and `ytdl-options.json`; complete native AdGuard Home binary, `AdGuardHome.yaml`, work directory, and root-owned `work/data`; the SSD-MINI monitor script, non-secret credential-field template, and LaunchAgent; the fixed LAN documentation server implementation and LaunchAgent; native UpSnap binary, wrapper, canonical plist, and persistent `pb_data` state; the update-report script and LaunchAgent; the homelab-repo Git snapshot repository (`/Users/patrykmac/homelab-repo`, including its `.git` history), its Forgejo deploy key, collector script, secret-scan gate library, allowlist, and watch-trigger generator script/plist; the homelab-docs-public-github push working copy (`/Users/patrykmac/homelab-docs-public-github`, including its `.git` history), its dedicated GitHub deploy key, push script, watch-path config, and watch-trigger generator script/plist; relevant LaunchAgents (including the UpSnap symlink-resolved plist) and the AdGuard LaunchDaemon; this document; backup/restore scripts; Homebrew and system reconstruction metadata; and included, excluded, and missing-path manifests.
- MeTube venv, Uptime Kuma `node_modules`, application repositories, and service binaries are deliberately included to speed recovery.
- Excluded: all iCloud Drive content, MeTube downloaded media, and the MeTube security-scoped bookmark (`/Users/patrykmac/Library/Application Support/MeTubeCleanupLauncher/metube-folder.bookmark`), which is non-portable and must be recreated interactively after restore; the SSD-MINI monitor's private Pushover credential file, runtime state, lock, and logs; LAN documentation viewer logs, bytecode cache, PID files, sockets, and temporary files; UpSnap transient logs (`/Users/patrykmac/homelab/upsnap/logs`), sockets, PID files, and temporary files; AI models/model weights and Ollama, MLX, Hugging Face, GGUF, and safetensors model files/caches; Time Machine data; Homebrew/package caches; temporary files; transient/rotated logs; crash reports; sockets, PID files, locks, and `.DS_Store`; staging data; and previous backup archives from staging.

### Uptime Kuma consistency

- The live database is never copied blindly. SQLite online `.backup` creates the staged `kuma.db`, then `PRAGMA integrity_check` validates it.
- Staged `kuma.db-wal` and `kuma.db-shm` are excluded. Restore verification fails if either appears in an archive.

### AdGuard privileged export

- Helper source: `/Users/patrykmac/homelab/backup/maintenance/export-adguard-data-root.sh`.
- Installed fixed-purpose helper: `/usr/local/sbin/export-macmini-adguard-data`.
- Restricted sudoers file: `/etc/sudoers.d/macmini-adguard-backup`.
- The helper copies the Mac mini's own root-owned `conf/AdGuardHome.yaml` (`root:wheel`, mode `0600`) and `work/data` into the controlled backup staging area, verifying the YAML's SHA-256 internally without logging the hash and leaving only the temporary staged copy readable by `patrykmac`. Both sources and allowed staging roots are fixed in the helper; it does not accept arbitrary source paths, grant general root access, stop/restart AdGuard, or expose configuration contents or secrets.
- Archive metadata records `full` or `partial-root-data-unavailable`; a complete run requires `full` and restore verification checks that declared mode.

### Homelab-repo Git snapshot coverage

- The complete `/Users/patrykmac/homelab-repo` directory, including `.git`, is staged verbatim (full commit history, not just the working tree), so the archive alone is enough to recover the repository.
- The Forgejo deploy private key `~/.ssh/forgejo-homelab-repo-mm-deploy` is staged with its mode preserved; restore verification explicitly checks that the staged copy is exactly mode `600` and fails the run otherwise.
- The collector, secret-scan gate library, allowlist, watch-trigger generator script, and the `com.patrykmac.homelab-repo-watch` plist are all staged, so the full automation — not just its output — can be reconstructed after a restore.
- Restore verification additionally runs `git fsck --full --strict` against the staged repository, in the same integrity-checking spirit as the existing Uptime Kuma SQLite `PRAGMA integrity_check`.

### Archive and verification

- Archive format: `macmini-homelab-YYYY-MM-DD_HHMMSS.tar.gz` with a matching `.sha256` file.
- Validation sequence: archive integrity test, local SHA-256 verification, NAS SHA-256 verification, non-restoring extraction/verification in a temporary directory, Uptime Kuma SQLite integrity validation, full AdGuard mode validation, homelab-repo Git integrity validation, and coverage checks. Verification never restores over live paths.

### Retention and NAS behavior

- Production retention is applied independently to `/Volumes/SSD-MINI/Homelab-Backups` and `/Volumes/macmini-backup-ssd/Homelab-Backups`: retain the newest five valid archive/checksum pairs, plus one valid pair closest to each of 7, 14, 21, 30, and 60 days old. Age-slot selections are additional to the newest five; an archive selected by multiple categories is kept once.
- Only a matching archive whose SHA-256 checksum validates is eligible. Unknown files and invalid or unpaired artifacts are not selected or deleted, and a sole valid pair is never pruned.
- Retention is invoked only after local archive and non-restoring verification succeed and NAS synchronization/checksum verification succeeds.
- Initialization marker: `/Users/patrykmac/homelab/backup/config/retention-initialized`. It was created after the first complete verified run; later runs use normal retention.
- Unknown files and invalid pairs are not deleted. Retention does not run after failed archive/verification/NAS synchronization, and the only valid backup is never deleted.
- The script detects an actual `smbfs` mount by the `macmini-backup-ssd` share name; it never trusts an empty local directory as NAS storage. NAS sync starts only after successful local verification. If NAS is unavailable, the valid local SSD archive remains intact and NAS failure never deletes it.

### Failure-only Pushover notifications

- Helper: `/Users/patrykmac/homelab/backup/lib/notify-pushover.sh`.
- Private credentials file: `/Users/patrykmac/homelab/backup/config/pushover.env`, mode `0600`.
- Exact title: `Mac Mini - backup status`.
- Notifications are sent only for `FAILED` or `DEGRADED` backup runs; status is in the message body. There are no routine success notifications. Delivery failure does not hide the original backup error, and credentials are never documented or logged.

### Manual commands

```sh
# Run a backup manually
/Users/patrykmac/homelab/backup/backup-macmini-homelab.sh

# Validate the newest local archive without restoring live paths
/Users/patrykmac/homelab/backup/restore-verify-macmini-homelab.sh

# Check the scheduled agent
launchctl print gui/$(id -u)/com.patrykmac.homelab-backup

# List backups
ls -lh /Volumes/SSD-MINI/Homelab-Backups
ls -lh /Volumes/macmini-backup-ssd/Homelab-Backups
```

## Service install policy

- Documentation-first approach for infrastructure changes.
- Do not reboot the Mac mini unless explicitly approved.
- Do not change router DNS during unrelated work.
- Do not touch Fedora miniPC services from this Mac mini documentation workflow.
- Do not install Docker Desktop unless explicitly approved.
- Prefer:
  - native `launchd` services
  - lightweight standalone binaries
  - PM2-style lightweight process management only if native `launchd` is not appropriate
- Avoid heavy background tooling and duplicate service stacks unless there is a clear backup/control need.

## Maintenance notes

- Keep this Mac mini minimal.
- Before adding new services, evaluate:
  - RAM impact
  - disk impact
  - whether the service competes with local LLM/OpenLLM use
  - whether a native install is possible
- Treat Fedora miniPC `HOMELAB.md` as the source of truth for the primary homelab.
- Use this file as the source of truth for the Mac mini backup/control node.
- After meaningful Mac mini homelab changes, update this file with:
  - service name
  - local URL/port
  - install path
  - whether it is native or containerized
  - whether it affects RAM/disk budget for local LLM use
  - backup/storage notes if relevant
- Preserve the distinction between intentional background services (LaunchAgents/LaunchDaemons) and unwanted restored GUI apps/windows after login.
