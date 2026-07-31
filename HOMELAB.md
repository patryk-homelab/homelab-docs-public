# Homelab Inventory

Last updated: 2026-07-31

This file documents the current miniPC Fedora homelab setup. It intentionally does not contain passwords, tokens, API keys, Uptime Kuma push URLs, or other secrets.

## How to open this file

- Canonical local path: `/home/patryk/docker/HOMELAB.md`
- SMB access path: `//192.168.10.12/documents/homelab/HOMELAB.md`
- Note: `/home/patryk/docker` itself is not exposed over SMB.

```bash
less ~/docker/HOMELAB.md
```

## Important IP addresses

- Router: Cudy WR3600
- Router LAN IP: `192.168.10.1`
- LAN subnet: `192.168.10.0/24`
- miniPC LAN IP: `192.168.10.12`
- Mac mini LAN IP: `192.168.10.13`
- miniPC mDNS hostname: `fedora.local`
- miniPC Tailscale IP: `100.118.164.107`
- miniPC Wake-on-LAN MAC: `68:1d:ef:4e:2e:a9`
- Synology NAS: DS223j
- Synology NAS IP: `192.168.10.92`
- NAS backup mount: `/mnt/nas-minipc-backup`
- NAS share: `//192.168.10.92/minipc-backup`

## Router / LAN recovery design

- Router is Cudy WR3600 at `192.168.10.1`.
- LAN subnet is `192.168.10.0/24`.
- miniPC Fedora is `192.168.10.12`.
- Synology NAS DS223j is `192.168.10.92`.
- miniPC Wake-on-LAN MAC is `68:1d:ef:4e:2e:a9`.
- Router and OpenVPN access must stay independent from miniPC, Tailscale, and AdGuard Home.
- Do not configure router or DHCP in a way that can accidentally remove emergency access to the router.

## OpenVPN on Cudy router

- OpenVPN Server runs on the Cudy WR3600 router.
- Purpose: emergency access to the home LAN and router panel even if miniPC, Tailscale, or AdGuard Home are down.
- Router panel over VPN: `http://192.168.10.1`
- NAS over VPN: `https://192.168.10.92:5001`
- miniPC over VPN, if the miniPC is running: `192.168.10.12`
- OpenVPN is critical as the management fallback when DNS or the miniPC fails.

## DNS policy

- DNS normal mode: DHCP DNS in the Cudy router now uses local AdGuard Home on the miniPC as primary DNS `192.168.10.12` and AdGuard Home on the Mac mini as secondary DNS `192.168.10.13`.
- `192.168.10.12` is the Fedora miniPC running local AdGuard Home DNS.
- `192.168.10.13` is the Mac mini running the secondary/fallback AdGuard Home DNS instance.
- Goal in normal mode: keep ad blocking available through mirrored AdGuard instances while preserving local LAN visibility on the primary miniPC resolver when clients use it.
- Important tradeoff: some clients may use both DHCP DNS servers in parallel instead of treating the second entry as strict failover, so both AdGuard instances should stay mirrored closely enough to avoid inconsistent filtering results.
- DNS maintenance mode: during Fedora/system update or reboot maintenance, the user may temporarily add a public resolver such as `1.1.1.1` as alternate DNS on the router.
- That temporary maintenance-mode DNS fallback is intentional and is not automatically a misconfiguration.
- Purpose of maintenance mode: keep internet access working if the miniPC or AdGuard Home is unavailable during update or reboot.
- Outside a planned maintenance window, do not leave a public DNS such as `1.1.1.1` configured next to the homelab AdGuard DNS entries longer than necessary because clients may use it in parallel and bypass AdGuard Home.
- After successful Fedora/system maintenance and verification, change router DHCP DNS back to primary `192.168.10.12` and secondary `192.168.10.13`.
- Emergency fallback is manual: connect through OpenVPN to the Cudy router, open the router panel, then change DHCP DNS from `192.168.10.12` to external DNS such as `1.1.1.1` / `8.8.8.8` or a verified AdGuard DNS Cloud resolver.
- Router and OpenVPN should remain independent from miniPC.

## AdGuard Home DNS behavior

- Local AdGuard Home runs on miniPC: `http://192.168.10.12:3002`, and fallback at `http://100.118.164.107:3002` over Tailscale.
- Router DHCP hands out `192.168.10.12` as primary DNS and `192.168.10.13` as secondary DNS.
- Because many clients may query both DHCP DNS servers, the Fedora and Mac mini AdGuard instances should remain mirrored for filters and upstream behavior where practical.
- When clients use DNS `192.168.10.12` through DHCP, AdGuard Home on the miniPC should see LAN clients as `192.168.10.x`.
- `100.x.x.x` addresses in AdGuard Home are most likely Tailscale clients.
- `172.20.0.1` is most likely Docker gateway or Docker-network traffic, not an unknown LAN device.

## Direct primary access and fedora.lan aliases

- Primary service access is direct LAN IP plus each service's verified published host port. `fedora.lan` is the Homarr dashboard alias and an additional LAN/Tailscale service alias.
- `fedora.lan` is resolved by the Fedora AdGuard Home DNS rewrite `fedora.lan -> 192.168.10.12`.
- Supported Caddy aliases are:
  - Homarr: `http://fedora.lan/`
  - Paperless: `http://fedora.lan/Paperless/`
- Caddy redirects `/Paperless` to `/Paperless/`. It accepts the canonical routes only for Host `fedora.lan`; arbitrary Host headers return `404`.
- Uptime Kuma 2.4.0 has no supported base-path setting: its HTML, dashboard redirect, assets, and Socket.IO endpoints are root-absolute.  Its supported administration URL remains `http://192.168.10.12:3001`; `/Kuma/` is deliberately not proxied so that it cannot be mistaken for a working login/WebSocket route.
- AdGuard Home has no supported web UI base-path setting: its login/UI/API paths are root-absolute.  Its supported administration URL remains `http://192.168.10.12:3002`; `/AdGuard/` is deliberately not proxied so that login/API behavior is not broken.
- Homarr application links use verified direct LAN URLs, including Paperless (`http://192.168.10.12:8010/Paperless/`).
- Paperless uses `PAPERLESS_URL=http://fedora.lan`, `PAPERLESS_FORCE_SCRIPT_NAME=/Paperless`, and allows `fedora.lan` plus the LAN IP. Direct `:8010` access remains same-origin and redirects only to its IP-based `/Paperless/` path.
- `fedora.local` is retained temporarily as a LAN-only compatibility alias for Homarr and lowercase Paperless routes. It is not canonical and must not be used for remote access.
- Tailscale remote access requires a tailnet split-DNS nameserver entry for domain `lan` pointing to `192.168.10.12`, while keeping MagicDNS enabled and global DNS unchanged.  The miniPC advertises the approved subnet route `192.168.10.0/24`, which remote clients need in order to reach that DNS server and HTTP service.
- The miniPC cannot safely create the tailnet split-DNS rule from its local CLI.  In the Tailscale admin console, open **DNS → Nameservers → Add nameserver**, enter `192.168.10.12`, select **Restrict to domain**, enter `lan`, save, and leave MagicDNS enabled.  Do not add it as a global nameserver.
- AdGuard returns `192.168.10.12` for `fedora.lan`; TCP/UDP DNS listens on `192.168.10.12:53`, Caddy listens on `192.168.10.12:80`, and the active and persistent Fedora firewall zones permit `dns`, `http`, and `mdns`. Remote-client DNS/HTTP verification remains pending the above admin-console split-DNS setting.

### Web-service access matrix

| Service | Container | Container port | Published host port / direct URL | `fedora.lan` alias |
|---|---|---:|---|---|
| Homarr | `homarr` | 7575 | `192.168.10.12:7575` / `http://192.168.10.12:7575/` | `http://fedora.lan/`; `https://m.patrykw.uk/` |
| SnapOtter | `snapotter` | 1349 | `192.168.10.12:1349` / `http://192.168.10.12:1349/` | `https://snapotter.patrykw.uk/`; internal DNS-01 TLS, LAN/Tailscale-only |
| Uptime Kuma | `uptime-kuma` | 3001 | `192.168.10.12:3001` / `http://192.168.10.12:3001/` | None; no reliable base-path support; `https://kuma.patrykw.uk/` |
| AdGuard Home UI | `adguardhome` | 3000 | `192.168.10.12:3002` / `http://192.168.10.12:3002/` | None; no reliable base-path support; `https://adguard.patrykw.uk/` |
| AdGuard DNS | `adguardhome` | 53 TCP/UDP | `192.168.10.12:53` | Provides the `fedora.lan` rewrite |
| Paperless-ngx | `paperless` | 8000 | `192.168.10.12:8010` / `http://192.168.10.12:8010/Paperless/` | `http://fedora.lan/Paperless/`; `https://paperless.patrykw.uk/Paperless/` |
| FreshRSS | `freshrss` | 80 | `192.168.10.12:8181` / `http://192.168.10.12:8181/` | None |
| Speedtest Tracker | `speedtest-tracker` | 80 | `192.168.10.12:8082` / `http://192.168.10.12:8082/` | None; `https://speedtest.patrykw.uk/` |
| changedetection.io | `changedetection` | 5000 | `192.168.10.12:5000` / `http://192.168.10.12:5000/` | None |
| Wallos | `wallos` | 80 | `192.168.10.12:8282` / `http://192.168.10.12:8282/` | None; LAN/Tailscale-only, not proxied by Caddy |
| CyberChef | `cyberchef` | 8080 | `192.168.10.12:8083` / `http://192.168.10.12:8083/` | None; `https://cyberchef.patrykw.uk/` |
| Stirling-PDF (ultra-lite) | `stirling-pdf` | 8080 | `192.168.10.12:8095` / `http://192.168.10.12:8095/` | Internal DNS-01 TLS: `https://pdf.patrykw.uk/`; LAN/Tailscale-only; no public proxy; ultra-lite has no authentication module |
| Watcharr | `watcharr` | 3080 | `192.168.10.12:8096` / `http://192.168.10.12:8096/` | Internal DNS-01 TLS: `https://seriale.patrykw.uk/`; LAN/Tailscale-only; authenticated; no public proxy |
| Czkawka | `czkawka` | 5800 | `192.168.10.12:8084` / `http://192.168.10.12:8084/` | None; LAN-only, no public exposure; ad-hoc/occasional use, not monitored |
| FileBrowser Quantum | `filebrowser` | 80 | `192.168.10.12:8085` / `http://192.168.10.12:8085/`; `100.118.164.107:8085` / `http://100.118.164.107:8085/` (Tailscale) | None; LAN + Tailscale only, full read-write access to `/home/patryk`; source `private: true` blocks public/anonymous shares; optional (not enforced) 2FA |
| RSS-Bridge | `rss-bridge` | 80 | `192.168.10.12:8086` / `http://192.168.10.12:8086/` | None; LAN-only, no public exposure; complements FreshRSS — generates feeds for FreshRSS to subscribe to |
| ConvertX | `convertx` | 3000 | `192.168.10.12:8094` / `http://192.168.10.12:8094/` | None; LAN/Tailscale-only through the approved subnet route; built-in login required, no public proxy |
| Shlink | `shlink` | 8080 | `192.168.10.12:8087` / `http://192.168.10.12:8087/` | Private admin/API: `https://shlink-admin.patrykw.uk/`; public redirect origin: `https://s.patrykw.uk/` after the pending Cloudflare Tunnel hostname is added |
| Shlink web client | `shlink-web-client` | 8080 | `192.168.10.12:8088` / `http://192.168.10.12:8088/` | Served only by internal DNS-01 Caddy at `https://shlink-admin.patrykw.uk/`; never routed through cloudflared |
| n8n | `n8n` | 5678 | `192.168.10.12:5678` / `http://192.168.10.12:5678/` | None |
| UpSnap | `upsnap` | 8090 | `http://192.168.10.12:8090/` | None; `https://upsnap.patrykw.uk/` |
| Caddy reverse proxy | `reverse-proxy` | 80 | `192.168.10.12:80` / `http://192.168.10.12/` (Homarr fallback) | Homarr, Paperless |
| Caddy (DNS-01 TLS) | `caddy` | 443 | `192.168.10.12:443` / 14 host routes under one wildcard cert `*.patrykw.uk` (see Web services / dashboards) | None; separate Caddy instance |

Containers without a published host port are internal-only dependencies and have no direct LAN URL: `paperless-db` (PostgreSQL 5432), `paperless-broker-1` (Redis 6379), and `diun`.

## WOL / recovery

- miniPC Wake-on-LAN MAC: `68:1d:ef:4e:2e:a9`.
- Fedora interface: `enp1s0`.
- Wake-on-LAN mode: magic packet. The NetworkManager connection `Wired connection 1` has persisted `802-3-ethernet.wake-on-lan=magic`; NetworkManager reapplies this property whenever the connection is activated, including after boot.
- Expected runtime NIC state is `Wake-on: g`. Unprivileged `ethtool enp1s0` cannot expose the Wake-on line because it returns a netlink permission error, so runtime `Wake-on: g` has not been independently observed.
- Waking from full shutdown also depends on a BIOS/firmware option such as Wake on LAN or Power On By PCIe. BIOS/firmware configuration cannot be authoritatively verified from Fedora, and waking from full shutdown has not been proven.
- After the next planned reboot, the operator must verify:

```bash
sudo ethtool enp1s0 | grep Wake-on
```

  Expected output must still include:

```text
Wake-on: g
```

- Recheck Wake-on-LAN after kernel, NIC-driver, NetworkManager, or firmware updates.
- Synology DS223j can send a Wake-on-LAN magic packet to the miniPC on the same LAN.
- OpenVPN to Cudy plus NAS WOL provides a recovery path when the miniPC is powered off but the router and NAS are still running.
- UpSnap on the miniPC provides a web UI to send Wake-on-LAN magic packets on the LAN. Its Mac mini device uses IP `192.168.10.13`, MAC `14:98:77:6b:d2:f9`, and netmask `255.255.255.0`, which resolves to broadcast `192.168.10.255` over UDP port `9`.
- The Apple Silicon Mac mini wakes from sleep only via WOL, never from a full shutdown. Recovery after power loss is handled by the Mac's autorestart setting, not WOL.
- A native UpSnap instance runs on the Mac mini at `http://192.168.10.13:8090` to wake the Fedora miniPC. The Homarr `UpSnap - Mac` tile is only a remote link to that native Mac mini service; no Mac mini service or configuration was changed from Fedora.

## Web services / dashboards

| Service | URL | Port | Type | Folder/config path | Backup covered | Notes |
|---|---|---:|---|---|---|---|
| Homarr | Primary `http://192.168.10.12:7575/`; Caddy LAN-IP fallback `http://192.168.10.12/`; alias `http://fedora.lan/`; DNS-01 TLS alias `https://m.patrykw.uk/` | Direct `7575`; Caddy `80`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/homarr/compose.yml`, `/home/patryk/docker/homarr/data` | Yes | Homarr `v1.71.0` is bound only to `192.168.10.12:7575`. Caddy routes the LAN-IP fallback and `fedora.lan` root to the same Homarr backend. `https://m.patrykw.uk/` is proxied by the separate internal-only DNS-01 TLS Caddy instance. No Docker socket is mounted. |
| Glances | Internal-only `http://glances:61208` from Homarr | No host port | Docker Compose | `/home/patryk/docker/glances/compose.yml` | Yes | Fedora miniPC CPU/RAM backend for Homarr. Image `nicolargo/glances:4.5.5`; connected only to `homarr_default`, no Docker socket, host mount, privileged mode, or runtime cache. |
| SnapOtter | `http://192.168.10.12:1349/`; internal DNS-01 TLS alias `https://snapotter.patrykw.uk/` | `1349`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/snapotter/compose.yml` | Yes — compose.yml + data + pgdata; redisdata intentionally excluded as non-essential cache | Self-hosted file/image/video/audio/PDF toolkit with local AI tools (background removal, upscaling, restoration, OCR, etc. — CPU-only, no external LLM/Ollama dependency). The HTTPS route is bound only to the miniPC LAN IP, so it is LAN/Tailscale-only with no public proxy. The default admin password is still flagged for change in the application and must be changed manually before treating this alias as safely complete. |
| FreshRSS | `http://192.168.10.12:8181` | `8181` | Docker Compose | `/home/patryk/docker/freshrss/compose.yml` | Yes | Data volumes: `freshrss_freshrss_data`, `freshrss_freshrss_extensions`. Working LAN browser URL is `http://192.168.10.12:8181/`. Working LAN mobile/API URL is `http://192.168.10.12:8181/api/greader.php`. Tailscale API URL `http://100.118.164.107:8181/api/greader.php` works only through Tailscale and should not be used for normal home LAN refresh. Refresh mechanism: in-container cron via `CRON_MIN=*/20`; default FreshRSS feed TTL is 1800 seconds / 30 minutes. Updates may still arrive in batches depending on feed cache state, upstream errors, or feed-specific failures. `IEEE Spectrum` (`https://spectrum.ieee.org/feeds/feed.rss`) should be reviewed separately if re-added. `/freshrss` is not currently reverse-proxied and should not be used unless a reverse-proxy route is added later. |
| AdGuard Home | `http://192.168.10.12:3002`; DNS-01 TLS alias `https://adguard.patrykw.uk/` | `3002`, DNS `53`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/adguardhome/compose.yml` | Yes | DNS bound to `100.118.164.107:53` and `192.168.10.12:53`. Router DHCP currently hands out `192.168.10.12` as primary DNS and `192.168.10.13` as secondary DNS; clients may use both, so the two AdGuard instances should stay mirrored. `https://adguard.patrykw.uk/` is proxied by the separate internal-only DNS-01 TLS Caddy instance. |
| Uptime Kuma | `http://192.168.10.12:3001`; DNS-01 TLS alias `https://kuma.patrykw.uk/` | `3001`; Caddy DNS-01 `443` | Docker | `/home/patryk/homelab/uptime-kuma/data` | Yes | Current data is a bind mount; backup script includes it under `bind_mounts/uptime_kuma_data/`. The backup push monitor exists as a separate push type. |

| UpSnap | `http://192.168.10.12:8090`; DNS-01 TLS alias `https://upsnap.patrykw.uk/` | `8090`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/upsnap/compose.yml`, `/home/patryk/docker/upsnap/data` | Yes | Wake-on-LAN web app (SvelteKit/Go/PocketBase), pinned to `ghcr.io/seriousm4x/upsnap:5.4.3`. It uses `network_mode: host`, but its web UI is bound only to `192.168.10.12:8090` through `UPSNAP_HTTP_LISTEN`, not exposed on Tailscale or `0.0.0.0`. Firewall `8090/tcp` is permanently allowed in the `enp1s0` zone. The admin account is managed in-app; DIUN tracks image updates. `https://upsnap.patrykw.uk/` is proxied by the separate internal-only DNS-01 TLS Caddy instance. |
| Speedtest Tracker | `http://192.168.10.12:8082`; DNS-01 TLS alias `https://speedtest.patrykw.uk/` | `8082`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/speedtest-tracker`, `/home/patryk/docker/speedtest-tracker/config` | Yes | SQLite, no external DB. Tracks home internet only, not Starlink truck internet. Schedule: 00:00, 06:00, 12:00, 18:00 daily. Homarr provides both its normal application link and native Internet Performance widget. `https://speedtest.patrykw.uk/` is proxied by the separate internal-only DNS-01 TLS Caddy instance. |
| changedetection.io | `http://192.168.10.12:5000` | `5000` | Docker Compose | `/home/patryk/docker/changedetection/compose.yml`, `/home/patryk/docker/changedetection/datastore` | Yes | Website and product price change monitoring. Notifications are configured manually in the UI through Pushover/Apprise; do not document or expose secret values. Homarr links to this service via LAN. |
| Wallos | `http://192.168.10.12:8282` | `8282` | Docker Compose | `/home/patryk/docker/wallos/compose.yml`, `/home/patryk/docker/wallos/db`, `/home/patryk/docker/wallos/logos` | Yes | LAN/Tailscale-only (not proxied by Caddy), OIDC intentionally left disabled due to known CVEs in Wallos's OIDC flow, personal subscription tracker. |
| NetAlertX | `http://192.168.10.12:8020/` | UI `8020`; GraphQL `8021` | Docker Compose | `/home/patryk/docker/netalertx/compose.yml`, `/home/patryk/docker/netalertx/data` | Yes | LAN device monitoring with pinned image `ghcr.io/netalertx/netalertx:26.7.1`, scanning only `192.168.10.0/24` on physical interface `enp1s0`. Server-side identification combines ARPSCAN, IPNEIGH (including NDP/IPv6) and NMAPDEV every five minutes; NMAPDEV host discovery retains ARP and adds ICMP echo, TCP SYN (22/80/443/3389/445/8080), TCP ACK (80/443), and UDP (53/67/123/161/5353) probes, with five retries and a 90-second host cap. ICMP status runs every five minutes; AVAHISCAN/mDNS, NSLOOKUP reverse DNS, DIG reverse DNS and NBTSCAN/NetBIOS are scheduled every five minutes with `REFRESH_FQDN=True`; VNDRPDT maintains the MAC vendor database. NMAP runs for newly found devices to collect ports and OS-fingerprint evidence without repeatedly scanning all devices. The application uses approved host networking and listens on `0.0.0.0:8020`; active and permanent firewalld rich rules accept `8020/8021` only from `192.168.10.0/24` and reject other sources, blocking Tailscale access. Homarr links directly to the LAN URL. Pushover credentials are configured securely in native NetAlertX settings; alerts are limited to genuinely new devices in `192.168.10.0/24`, and secret values are not documented. The `INTRNT` plugin shows the public Internet IP, which is expected behavior. No router, AdGuard, or other external-service importer is integrated; no reverse proxy, DNS rewrite, or Tailscale service access is configured. |
| CyberChef | `http://192.168.10.12:8083`; DNS-01 TLS alias `https://cyberchef.patrykw.uk/` | `8083`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/cyberchef/docker-compose.yml` | Yes | Stateless local data conversion and analysis tool. Bound only to the miniPC LAN IP, not publicly exposed or proxied; Homarr links to it directly. `https://cyberchef.patrykw.uk/` is proxied by the separate internal-only DNS-01 TLS Caddy instance. |
| Stirling-PDF (ultra-lite) | Direct `http://192.168.10.12:8095/`; internal DNS-01 TLS alias `https://pdf.patrykw.uk/` | `8095` (container `8080`); Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/stirling-pdf/compose.yml`, `/home/patryk/docker/stirling-pdf/configs` | Yes | Pinned to `stirlingtools/stirling-pdf:2.14.2-ultra-lite`, chosen for its smaller footprint and lack of heavy external dependencies. It retains core PDF operations such as merge, split, rotate, page organization, image/PDF basics, passwords, metadata, watermarking, and signing. Compared with full Stirling-PDF, ultra-lite omits OCR; Office/common-file conversion; PDF-to-Word/Presentation/XML/HTML/PDF-A; HTML/URL/EML/ebook/vector conversions; compression and repair backends; scan-image extraction; and other tools requiring LibreOffice, Python/OpenCV, Tesseract/OCRmyPDF, QPDF, Ghostscript/ImageMagick, WeasyPrint, Calibre, or related binaries. The deployed ultra-lite JAR reports `activeSecurity=false` and ignores login settings because the authentication module is not included, so any client with LAN or approved Tailscale subnet-route access can use it. It binds only to `192.168.10.12`, has no public Caddy/cloudflared route, uses the dedicated `stirling_pdf_internal` network, drops all capabilities before adding only `SETUID` and `SETGID` for the entrypoint's root-to-`stirlingpdfuser` transition, uses `no-new-privileges`, and has no Docker socket or privileged mode. A two-input PDF merge was verified end to end. The generic backup captures `compose.yml`; generated settings under `configs/` are archived as `bind_mounts/stirling_pdf_configs/` and required by restore verification. After the JVM was observed accepting connections but returning no bytes on both direct and Caddy paths, its thread dump showed all four virtual-thread carrier workers stalled during first-request CORS/class loading, including a wait on Spring Boot's nested-JAR lock. Compose now sets `JAVA_CUSTOM_OPTS=-Dspring.threads.virtual.enabled=false`, using Stirling's supported option hook to keep HTTP handling on platform threads; no Caddy header, WebSocket, host, origin, or base-URL override is needed. The container health check, direct URL, proxied health API, and identical direct/proxied HTML all return HTTP 200 after recreation. Fedora AdGuard resolves `pdf.patrykw.uk` correctly to `192.168.10.12`; the Mac mini AdGuard currently returns the incorrect `192.168.10.13` and must be corrected manually to `192.168.10.12`. |
| Watcharr | Direct `http://192.168.10.12:8096/`; internal DNS-01 TLS alias `https://seriale.patrykw.uk/` | `8096` (container `3080`); Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/watcharr/compose.yml`, protected `/home/patryk/docker/watcharr/.env`, `/home/patryk/docker/watcharr/data` | Yes | Pinned to `ghcr.io/sbondco/watcharr:v3.0.1`. A personal TMDB API key is optional in this release: `TMDB_KEY` is intentionally unset and Watcharr's bundled TMDB access successfully returned metadata. The initial admin account was created through the setup API; further signup is disabled. Patryk subsequently changed the password in-app. The generated username/password retained in mode-600 `.env` are superseded historical bootstrap records, are not valid current login credentials, and are not injected into the container; the current account state lives only in persistent `data/watcharr.db`. `data/watcharr.json` holds server configuration. The container runs as UID/GID 1000, binds only to `192.168.10.12`, drops all capabilities, uses `no-new-privileges`, a read-only root filesystem, narrow `/tmp` tmpfs, and the dedicated `watcharr_internal` bridge; it has no Docker socket or privileged mode. There is no cloudflared/public exposure. Login, metadata search, and adding the real test entry `Fight Club` (TMDB 550, status `PLANNED`) were verified end to end. Backup captures Compose, protected `.env`, `CREDENTIALS.md`, and a consistent full `data` snapshot under `bind_mounts/watcharr_data/`; restore verification requires both SQLite and server config. Fedora AdGuard resolves `seriale.patrykw.uk` correctly to `192.168.10.12`; the Mac mini AdGuard currently returns the incorrect `192.168.10.13` and must be corrected manually to `192.168.10.12`. |
| Czkawka | `http://192.168.10.12:8084/` | `8084` (container `5800`) | Docker Compose | `/home/patryk/docker/czkawka/compose.yml`, `/home/patryk/docker/czkawka/config` | Yes | Occasional/ad-hoc duplicate-file finder, pinned to `jlesage/czkawka:v26.07.2`, bound only to the miniPC LAN IP with no Caddy route, `fedora.lan` alias, DNS-01 TLS hostname, or Cloudflare tunnel. `/home/patryk` is mounted as `/storage:ro`, so the container cannot modify host files. Not monitored by Uptime Kuma and intentionally excluded from the Daily Homelab Report. |
| FileBrowser Quantum | `http://192.168.10.12:8085/`; `http://100.118.164.107:8085/` (Tailscale) | `8085` (container `80`) | Docker Compose | `/home/patryk/docker/filebrowser/compose.yml`, protected `/home/patryk/docker/filebrowser/.env`, `/home/patryk/docker/filebrowser/data` | Yes | Pinned to `gtstef/filebrowser:1.5.0-stable`; the sole intentionally full read-write service for `/home/patryk`, including administrative paths and secrets, as an informed LAN + Tailscale-only choice (not public Internet exposure). It runs as `filebrowser` (1000:1000), drops all capabilities, and uses `no-new-privileges`. Its private `Home` source disables public/anonymous shares; password sign-in is enabled, sign-up is disabled, and 2FA is available but not enforced. |
| RSS-Bridge | `http://192.168.10.12:8086/` | `8086` (container `80`) | Docker Compose | `/home/patryk/docker/rss-bridge/compose.yml`, `/home/patryk/docker/rss-bridge/config` | Yes | Pinned to `rssbridge/rss-bridge:sha-fd17579` (matched the `latest` digest at setup time). Generates RSS/Atom/JSON feeds for sites without native feeds; feed URLs are manually added as sources inside FreshRSS by the operator — no backend integration between the two services. `cap_drop: ALL` plus a minimal `cap_add` (`CHOWN`, `SETUID`, `SETGID`, `NET_BIND_SERVICE`, `DAC_OVERRIDE`) and `no-new-privileges` are applied; the image has no PUID/GID support and its nginx/php-fpm masters must start as root before dropping to `www-data`, so the container cannot run fully unprivileged. All bundled bridges are enabled (`enabled_bridges[] = *`); authentication remains disabled, consistent with the LAN-only, unauthenticated trust model already used for CyberChef. Bound only to the miniPC LAN IP, no Caddy route, `fedora.lan` alias, or public exposure. |
| ConvertX | `http://192.168.10.12:8094/`; health `http://192.168.10.12:8094/healthcheck` | `8094` (container `3000`) | Docker Compose | `/home/patryk/docker/convertx/compose.yml`, protected `/home/patryk/docker/convertx/.env`, `/home/patryk/docker/convertx/CREDENTIALS.md`, `/home/patryk/docker/convertx/data` | Yes | Pinned to `ghcr.io/c4illin/convertx:v0.18.0`; never deploy below `v0.18.0`. This floor is past the `v0.16.0` arbitrary-file-write fix for CVE-2025-66449 and the `v0.17.0` arbitrary-file-deletion path-traversal fix for CVE-2026-24741. Built-in email/password authentication is required: `ALLOW_UNAUTHENTICATED=false` and further account registration is disabled. Automation created the initial account, after which Patryk changed its email and password in-app. Current login state lives only in `data/mydb.sqlite`; the original `CONVERTX_ADMIN_EMAIL` and `CONVERTX_ADMIN_PASSWORD` values in `.env` are superseded historical bootstrap records, are not injected into the container, and must not be treated as current credentials or used to overwrite SQLite. `JWT_SECRET` remains the only runtime secret loaded from `.env`; `CREDENTIALS.md` records this boundary without containing current credentials. `HTTP_ALLOWED=true` is limited to this direct trusted-LAN HTTP deployment and must not accompany public exposure. The container binds only to `192.168.10.12`, uses the dedicated `convertx_internal` network, drops all capabilities, uses `no-new-privileges`, has a read-only root filesystem with narrow converter tmpfs paths, and has no Docker socket or privileged mode. Its root-owned mode-750 data directory holds SQLite state, uploads, outputs, and history and is archived through a short-lived root backup container. No cloudflared, public Caddy DNS-01, DNS rewrite, or firewalld change is configured because this file-handling service and its recent vulnerability history warrant LAN/Tailscale-only access; tailnet access uses the approved LAN subnet route. The Homarr app uses the closest available official dashboard-icons file-conversion icon (`convertio.svg`) because the set has no ConvertX icon. |
| Shlink | Private admin and REST API `https://shlink-admin.patrykw.uk/`; direct LAN server `http://192.168.10.12:8087/`; direct LAN web client `http://192.168.10.12:8088/`; filtered public origin `http://192.168.10.12:8092/`; intended public redirects `https://s.patrykw.uk/` | Shlink `8087`; web client `8088`; public gateway `8092`; internal Caddy `443` | Docker Compose | `/home/patryk/docker/shlink/compose.yml`, protected `/home/patryk/docker/shlink/.env`, `/home/patryk/docker/shlink/postgres-data`, `/home/patryk/docker/shlink/Caddyfile.public`, `/home/patryk/docker/shlink/Dockerfile.public` | Yes | Shlink `5.1.5`, web client `4.8.0`, PostgreSQL `17.10`, and Caddy gateway `2.11.4` use only the dedicated `shlink_internal` bridge and are not attached to any other stack. All containers drop every capability, use `no-new-privileges`, and have read-only root filesystems with narrow tmpfs/runtime and database-data writes. Shlink and the web client bind only to `192.168.10.12`; PostgreSQL has no host port. The public gateway accepts only GET/HEAD, returns `404` for `/rest` and `/rest/*`, and has no route to the web-client container, so public link creation and administration are structurally unavailable. The no-role admin API key was generated through the local Shlink CLI and is stored only in mode-600 `.env`; the client is not preconfigured with it. Upstream explicitly does not support SQLite for production, so PostgreSQL is used despite the extra container. GeoLite2 is intentionally disabled to avoid a MaxMind account/license secret and external dependency. `shlink-admin.patrykw.uk -> 192.168.10.12` still needs to be added manually as the single new AdGuard Home DNS rewrite. The remotely managed Cloudflare Tunnel still needs a manual `s.patrykw.uk -> http://192.168.10.12:8092` public-hostname entry before Internet redirects are live; the local DNS token cannot edit tunnel configuration. No firewalld change is needed because the active FedoraWorkstation zone already allows HTTPS and the high LAN-bound ports. |
| Fedora Web Console / Cockpit | `https://192.168.10.12:9090/`, hostname/LAN alternative `https://fedora:9090/` | `9090` | Fedora system service | System package / Cockpit service | No | Fedora Web Console for host administration. Homarr uses the IP-based HTTPS URL. The Cockpit service is intentionally not monitored by Uptime Kuma but remains active and accessible. |
| n8n | `http://192.168.10.12:5678` | `5678` | Docker Compose | `/home/patryk/docker/n8n/docker-compose.yml`, `/home/patryk/docker/n8n/.env`, `/home/patryk/docker/n8n/data` | No | Local automation/orchestration for homelab workflows. Not publicly exposed. Contains two active workflows: Disk free space alert and Homelab Status API. Note: encryption key is stored securely in `.env`. n8n is local-only over HTTP. N8N_SECURE_COOKIE=false is intentionally set because n8n is accessed over LAN HTTP, not HTTPS. Do not expose n8n publicly with this setting. |
| Paperless-ngx | Primary `http://192.168.10.12:8010/Paperless/`; alias `http://fedora.lan/Paperless/`; DNS-01 TLS alias `https://paperless.patrykw.uk/Paperless/` | Direct `8010`; Caddy alias `80`; Caddy DNS-01 `443` | Docker Compose | `/home/patryk/docker/paperless/docker-compose.yml`, `/home/patryk/docker/paperless/.env`, `/home/patryk/docker/paperless/pgdata`, `/home/patryk/docker/paperless/data`, `/home/patryk/Documents/paperless/media` | Yes | Document management.  `PAPERLESS_FORCE_SCRIPT_NAME=/Paperless` means direct root redirects to its direct-IP `/Paperless/` path; it never redirects to `fedora.lan`. PostgreSQL and Redis have no published ports. The PostgreSQL DB password now lives in `/home/patryk/docker/paperless/.env` (referenced via `${POSTGRES_PASSWORD}` substitution) and is no longer hardcoded in `docker-compose.yml`. Consume and export folders are excluded from backup. `PAPERLESS_ALLOWED_HOSTS` in `/home/patryk/docker/paperless/.env` includes `paperless.patrykw.uk`, and `PAPERLESS_CSRF_TRUSTED_ORIGINS=https://paperless.patrykw.uk` is set, so the DNS-01 TLS alias works alongside the existing LAN URLs. |
| Reverse proxy | LAN-IP Homarr fallback `http://192.168.10.12/`; aliases `http://fedora.lan/` and `/Paperless/` | `80` bound to `127.0.0.1` and `192.168.10.12` | Docker Compose / Caddy container | `/home/patryk/docker/reverse-proxy/compose.yml`, `/home/patryk/docker/reverse-proxy/Caddyfile` | Yes | Caddy routes Homarr at the root and retains the Paperless subpath with its trailing-slash redirect. It passes the normal reverse-proxy Host and client headers and returns 404 for unrelated hosts. |
| Caddy (DNS-01 TLS) | 14 host routes (see below), one wildcard cert `*.patrykw.uk` | `443` bound only to `192.168.10.12` | Docker Compose / custom Caddy build | `/home/patryk/docker/caddy/compose.yml`, `/home/patryk/docker/caddy/Caddyfile`, `/home/patryk/docker/caddy/Dockerfile`, `/home/patryk/docker/caddy/.env` | Yes | Separate Caddy instance (`caddy:2-builder` + `caddy-dns/cloudflare` via `xcaddy`) that obtains one wildcard Let's Encrypt certificate (`*.patrykw.uk`) through ACME DNS-01 against the Cloudflare-managed `patrykw.uk` zone, for internal-only (LAN/Tailscale) access to services that need trusted, non-self-signed TLS. Single `*.patrykw.uk` site block in the Caddyfile with `host`-matched `handle` blocks per hostname, routing to: `kuma.patrykw.uk` -> 192.168.10.12:3001, `m.patrykw.uk` -> 192.168.10.12:7575, `adguard.patrykw.uk` -> 192.168.10.12:3002, `paperless.patrykw.uk` -> 192.168.10.12:8010, `speedtest.patrykw.uk` -> 192.168.10.12:8082, `cyberchef.patrykw.uk` -> 192.168.10.12:8083, `snapotter.patrykw.uk` -> 192.168.10.12:1349, `pdf.patrykw.uk` -> 192.168.10.12:8095, `seriale.patrykw.uk` -> 192.168.10.12:8096, `upsnap.patrykw.uk` -> 192.168.10.12:8090, `shlink-admin.patrykw.uk` -> Shlink REST API on 192.168.10.12:8087 only for `/rest`/`/rest/*` and the separate static web client on 192.168.10.12:8088 for all other paths, `adguard-mm.patrykw.uk` -> 192.168.10.13:3002, `kuma-mm.patrykw.uk` -> 192.168.10.13:3003, `metube.patrykw.uk` -> 192.168.10.13:8091. The three `-mm` hostnames proxy over LAN to Mac mini services documented in the Mac mini's own `HOMELAB-MACMINI.md`, not duplicated here. No port 80 is published; DNS-01 does not require public reachability, and no hostname is publicly exposed by this Caddy. The `pdf` and `seriale` routes validate and serve the correct Fedora applications. Fedora AdGuard has both rewrites pointing to `192.168.10.12`; read-only queries to the Mac mini AdGuard currently return `192.168.10.13` for both names, which is incorrect because that host has no listener for these routes. Correct both Mac mini AdGuard rewrites manually to `pdf.patrykw.uk -> 192.168.10.12` and `seriale.patrykw.uk -> 192.168.10.12`; clients may query either resolver. Independent of, and does not replace, the plain-HTTP `reverse-proxy` Caddy instance above. `.env` holds `CLOUDFLARE_API_TOKEN` and is backed up but not committed to `homelab-repo`. |
| Forgejo | `http://192.168.10.12:3300/` (HTTP web UI); Git-over-SSH `192.168.10.12:2222` | `3300`, SSH `2222` | Docker Compose | `/home/patryk/docker/forgejo/compose.yml`, `/home/patryk/docker/forgejo/data` | Yes | Self-hosted Git server (`codeberg.org/forgejo/forgejo:16.0.1`), SQLite backend, new-user registration disabled (`FORGEJO__service__DISABLE_REGISTRATION=true`). Hosts `homelab-repo`'s one-way snapshot push over SSH via a dedicated deploy key. Reachable via both LAN and Tailscale by default, per the FedoraWorkstation firewalld zone's stock behavior; no restrictive rich rule is configured for it, unlike NetAlertX. See Infrastructure services and Backup coverage. |
| Synology DSM | `https://192.168.10.92:5001` | `5001` | External NAS | Synology DSM | No | Not hosted on miniPC. NAS stores backup copies. |
| Tailscale Admin Console | `https://login.tailscale.com/admin/machines` | HTTPS | External SaaS | Tailscale account | No | Useful for tailnet administration, not hosted locally. |

## Homarr dashboard

- Homarr `v1.71.0` is the approved primary miniPC dashboard. Its Compose file is `/home/patryk/docker/homarr/compose.yml`; all recoverable state is under `/home/patryk/docker/homarr/data`, with the encryption secret protected in `/home/patryk/docker/homarr/.env`.
- Homarr includes one operational Synology DiskStation integration named `Synology NAS`, using DSM URL `https://patrykwo36.synology.me:5001` and DSM service account `homarr-monitor`. The account is limited to DSM access but is a member of the DSM administrators group as required by the integration. The local DNS rewrite is `patrykwo36.synology.me -> 192.168.10.92`. Homarr stores the credentials securely; they are not documented here. The dashboard widgets display CPU usage, memory usage, NAS temperature, volume status, and used and available storage; `volume_1` reports status `normal`. No further configuration is currently required.
- The same private board is available at `http://192.168.10.12:7575/` and `http://fedora.lan/`. Caddy proxies the latter to the direct LAN-bound Homarr listener; both origins use the same database, board, app records, widget layout, integration data, and local assets. Login cookies are origin-specific, so a user may need to authenticate once per origin.
- The board currently has five named sections: Monitoring, Apps, Management, Documentation, and Mac Mini. Item membership and coordinates are maintained independently in each layout as described below.
- Homarr stores dashboard placement independently for the custom `iPhone` mobile layout (3 columns, breakpoint 0) and the `Other` desktop layout (10 columns, breakpoint 600). The SQLite `item_layout` and `section_layout` records are keyed by `layout_id`, so dragging a tile in one layout does not move it in the other. The authenticated MCP connector can create an item only in the default empty section and cannot read or move existing items, but Homarr's authenticated tRPC `board.getBoardByName` / `board.saveBoard` pair exposes the complete validated board payload, including layout-specific positions.
- `/home/patryk/scripts/homarr-place-item.py` is the supported local cross-layout placement helper. It resolves an existing app/item by exact name or item ID, finds a named dynamic section, selects the first collision-free position in that section on **every current or future named layout**, preserves the complete existing item and section arrays required by `board.saveBoard`, aborts if the board changes between planning and saving, and verifies after the write that all unrelated items, sections, and layouts remained identical. The default is a read-only dry run; repeat the same command with `--apply` only after reviewing its plan:

  ```bash
  /home/patryk/scripts/homarr-place-item.py --board dashboard --item "RSS-Bridge" --section Apps
  /home/patryk/scripts/homarr-place-item.py --board dashboard --item "RSS-Bridge" --section Apps --apply
  ```

  Authentication can come from `HOMARR_API_KEY`, `HOMARR_API_KEY_FILE`, `--api-key-file`, or the existing Codex Homarr credential; the credential is never printed. The helper moves only an already-existing board item. It deliberately does **not** create or delete apps/items/layouts/sections, resize a section, reflow unrelated elements, or force a placement when the target section is full. Direct SQLite placement edits remain prohibited.
- **Deletion gotcha:** one Homarr item is shared across all layouts. Deleting a tile while viewing any one layout deletes that shared item everywhere, not just from the current layout. An unsorted or misplaced tile must be dragged into place, left as-is, or moved with `homarr-place-item.py`; never delete it merely to fix organization on one layout.
- Current verified placement state: the `iPhone` Apps section is `3x4` (expanded from `3x3` through a disposable-board tRPC test and verified production save); its `Other` layout remains `3x4` and was untouched. The six previously restored application tiles — CyberChef, FreshRSS, n8n, Paperless, UpSnap, and Shlink — plus `ConvertX` are correctly placed on both layouts. Czkawka, Filebrowser, RSS-Bridge, Stirling-PDF, and Watcharr are correctly placed in Apps on both layouts without overlap. Two production reads eight seconds apart were identical, confirming that the prior concurrent-save issue is resolved. The supported placement helper still deliberately does not resize or reflow sections.
- The obsolete `Homelab Docs Public` and `Homelab Docs Public - Mac` app records and their former `docs.patrykw.uk` links are not part of the board or app catalog. The authenticated Homarr data audit contains no remaining `docs.patrykw.uk` app, item, widget, integration, or layout reference. The separate `GitHub` tile for the public documentation repository remains.
- Czkawka is an occasional duplicate-file finder at `http://192.168.10.12:8084/`. Its Homarr app ID is `pvwy1xt5nzldec85tskqstb2` and current dashboard item ID is `h4ibzp3mfj9fkb834ngciow6`; it uses the official `czkawka.svg` icon. It is in Apps on both `iPhone` and `Other`.
- Filebrowser is a FileBrowser Quantum link at `http://192.168.10.12:8085/`. Its Homarr app ID is `iln0ehbxqlrp9jyr8b9naazm` and current dashboard item ID is `q3q4m46gcrdnvvjvqpc5s2jm`; it uses the official `filebrowser-quantum.svg` icon. It is in Apps on both `iPhone` and `Other`.
- The following application records were originally created through the authenticated connector. Their stored placement is layout-specific under the rule above.
  - `UpSnap` is the Fedora miniPC UpSnap instance: URL `http://192.168.10.12:8090`; health URL `http://192.168.10.12:8090/api/health`; Homarr app ID `qu97k9v530nicivda6v5omuc`; dashboard item ID `e66ixcjo1ocghaeaoxjq1u5y`; official Homarr icon `upsnap.svg`; intended section Management. This tile links to the miniPC UpSnap instance.
  - `UpSnap - Mac` is the Mac mini native UpSnap instance: URL `http://192.168.10.13:8090`; health URL `http://192.168.10.13:8090/api/health`; Homarr app ID `bi6ndnndxytrfjqmzt3xnc06`; dashboard item ID `o753cbfpf27lcmb7pgpn2g9n`; official Homarr icon `upsnap.svg`; intended section Mac Mini. This tile is only a remote link to the native Mac mini UpSnap service.
  - `Filebrowser - Mac` is the Mac mini FileBrowser Quantum instance: URL `http://192.168.10.13:8085/`; health URL is its root URL; Homarr app ID `to58wou2rh2t8s22exiylkb4`; current dashboard item ID `i9914hjq90rrxoh59m00vkzl`; official Homarr icon `filebrowser-quantum.svg`. It is in Mac Mini on both `iPhone` and `Other`. This tile is only a remote link to the native Mac mini service.
  - `Forgejo` is the self-hosted Git server: URL `http://192.168.10.12:3300`; health URL `http://192.168.10.12:3300/api/healthz`; Homarr app ID `qqsjtxza2cp20lohtz13d5ak`; dashboard item ID `gyfipchd9tifx2s2kxirustd`; official Homarr icon `forgejo.svg`; intended section Management. This tile links to the Forgejo web UI.
  - `Shlink` is the URL-shortener administration web client: URL `https://shlink-admin.patrykw.uk/`, used because the private Caddy hostname serves the web client over trusted TLS on LAN/Tailscale while keeping the public `s.patrykw.uk` redirect origin separate; health URL `https://shlink-admin.patrykw.uk/rest/health`; Homarr app ID `ev8chpc3fm5zy05pbu0ikbcu`; dashboard item ID `w3ianl2ny2cq8zcmhjquja6p`; official Homarr icon `shlink.svg`; intended section Management. This tile was auto-placed in Homarr's unnamed `empty` default section and still needs to be manually dragged into Management.
  - `GitHub` is the public homelab documentation repository: URL `https://github.com/patryk-homelab/homelab-docs-public`; no health check; Homarr app ID `hyresgcz6nf88md3yiyy1ebt`; dashboard item ID `d6zqh6t737ghi9p3cdkvw8gc`; official Homarr icon `github.svg`; intended section Management. This tile links to the public homelab documentation repository.
  - `Czkawka` is the occasional duplicate-file finder: URL `http://192.168.10.12:8084/`; no health check; Homarr app ID `pvwy1xt5nzldec85tskqstb2`; current dashboard item ID `h4ibzp3mfj9fkb834ngciow6`; official Homarr icon `czkawka.svg`. It is in Apps on both `iPhone` and `Other`.
  - `ConvertX` is the authenticated LAN/Tailscale-only file converter: URL `http://192.168.10.12:8094/`; health URL `http://192.168.10.12:8094/healthcheck`; Homarr app ID `j10wy9dvjpn22vz30ldwszzo`; dashboard item ID `wj0b25mehn94k3srjizysu4x`; closest available official dashboard-icons file-conversion icon `convertio.svg` because no ConvertX icon exists. It is in Apps on both `iPhone` and `Other`.
- `RSS-Bridge` is the LAN-only feed-generator link: URL `http://192.168.10.12:8086/`; health URL `http://192.168.10.12:8086/` (root returns HTTP 200; `/health` returns 404). Its Homarr app ID is `emputz17z1pa2ipemfjfrdrt` and current dashboard item ID is `us9y4swkioyceyqzke3nfb90`; it uses Homarr's official `rss-bridge.svg` icon. It is in Apps on both `iPhone` and `Other`.
- `Stirling-PDF` is the LAN/Tailscale-only PDF-tools link: URL `http://192.168.10.12:8095/`; health URL `http://192.168.10.12:8095/api/v1/info/status`; Homarr app ID `jajq89axhanpyfqibmqagsmj`; current dashboard item ID is `fmrpim1t5zvnarce6yxc2ocm`; official dashboard-icons file `stirling-pdf.svg`; intended section Apps. It is in Apps on both `iPhone` and `Other`.
- `Watcharr` is the authenticated LAN/Tailscale-only movie/TV tracking link: URL `http://192.168.10.12:8096/`; health URL `http://192.168.10.12:8096/api/auth/available`; Homarr app ID `jgw110vjlvsdtdmdwbrar6ht`; current dashboard item ID is `rpccnjc1zonbdk9udp0v97bf`; official dashboard-icons file `watcharr.svg`; intended section Apps. It is in Apps on both `iPhone` and `Other`.
- `Installed Services` is a native Homarr `notebook` widget containing one manually curated informational list of the active Fedora miniPC and Mac mini homelab services/automations that have no human-facing web UI. Its Homarr dashboard item ID is `er6cl8t0pu6stcu4l0p8mz5q`; it was auto-placed at the connector's default `1x1` size in Homarr's unnamed `empty` section. Its intended section is Management, alongside the other backend/admin-facing entries. The connector cannot safely place or resize an item inside an existing section, so the user must manually drag it into Management and resize it to a readable footprint, like the other automatically placed items documented above.
- The `Installed Services` list is manually curated and must be reviewed whenever a no-UI service or automation is added to or removed from either host. Apply the existing [Maintenance rule](#maintenance-rule) when updating the current-state inventory; do not let this widget become a second source of truth.
- Native widgets are below the dashboard groups: the Speedtest Tracker widget is on the left and the System Resources widget is beside it. The normal clickable Speedtest application link remains separate from the genuine native Speedtest Tracker widget, which is backed by the `Internet Performance` integration and displays download, upload, and ping.
- Fedora miniPC CPU/RAM monitoring is provided by Glances `4.5.5` at `/home/patryk/docker/glances/compose.yml`. It is internal-only on the existing `homarr_default` Docker bridge as `http://glances:61208`, has no published host port, no Docker socket, no host mount, no privileged mode, and drops all Linux capabilities. The documented `pid: host` mode is used only so Glances reads Fedora host CPU/RAM rather than container values.
- The Fedora miniPC native Homarr System Resources widget uses the `Fedora miniPC - Glances` integration. It displays CPU and Memory only; Network is disabled, and disk, Docker containers, processes, GPU, and all other metrics are disabled or absent. It is placed beside the Speedtest Tracker widget.
- A separate native System Resources widget labelled `Mac mini` is immediately to the right of the Fedora widget in the same lower native-widget row. It uses the `Mac mini - Glances` integration at `http://192.168.10.13:61208` and displays CPU and Memory only. It depends on the native Mac mini Glances LaunchAgent remaining active; this endpoint is LAN-only, is not published by Fedora Docker, and is not routed through Caddy.
- Basic Mac mini widget validation/troubleshooting from Fedora:

```bash
curl -fsS http://192.168.10.13:61208/api/4/cpu
curl -fsS http://192.168.10.13:61208/api/4/mem
docker exec homarr node -e 'fetch("http://192.168.10.13:61208/api/4/cpu").then(r => r.json()).then(console.log)'
docker logs --tail=80 homarr
```
- The dark persistent star field is a native board background: a local embedded sparse SVG pattern with a dark navy base, fixed attachment, repeat, and no animation. It does not depend on an external image. Cards remain opaque for readability.
- Homarr has no Docker socket mount and Docker discovery is not enabled. The Mac mini is represented only by the four existing application links.
- Active backup coverage contains Homarr's Compose file, protected `.env`, complete `data` tree, SQLite database, Redis state, trusted certificates, assets, board/app/item/widget/integration state, and authentication data.

## Uptime Kuma and Monitoring Architecture

Cross-monitoring is intentionally minimal and bidirectional between the Fedora miniPC and the Mac Mini.
Both instances have Pushover notifications configured.

### Mac Mini Uptime Kuma
- URL: http://192.168.10.13:3003
- Version: 2.4.0
- Installation: native macOS installation (no Docker Desktop)
- LaunchAgent: com.patrykmac.uptime-kuma
- Database: SQLite database
- Status: Installed and active
- Active monitors (monitoring Fedora):
  1. AdGuard - MiniPC (DNS resolver: 192.168.10.12:53)
  2. MiniPC - Uptime Kuma (URL: http://192.168.10.12:3001)
  3. MiniPC Ping (host: 192.168.10.12)

### Fedora miniPC Uptime Kuma
- URL: http://192.168.10.12:3001
- Active monitors (monitoring Mac Mini):
  1. AdGuard - Mac Mini (DNS resolver: 192.168.10.13:53)
  2. Mac Mini - Uptime Kuma (URL: http://192.168.10.13:3003)
  3. Mac Mini Ping (host: 192.168.10.13)
- Existing active monitors (local/misc):
  - FreshRSS
  - Homelab Backup
  - NAS Mount
  - NAS Ping
  - Tailscale MiniPC

### Omitted monitor
- **miniPC Cockpit**: The "miniPC Cockpit" monitor is intentionally omitted. The Fedora Web Console / Cockpit service itself remains installed, active, and accessible at https://192.168.10.12:9090/.

## n8n Workflows

There are exactly two active n8n workflows:

- **Workflow name**: Disk free space alert
  - **Status**: active / verified manually in UI
  - **Schedule**: every 6 hours
  - **Checks**: filesystem containing `/home/patryk` via read-only bind mount `/host/home/patryk`. Disk free space alert uses n8n Code node with `fs.statfsSync`, not Execute Command.
  - **Threshold**: alert when free space < 200 GB
  - **Alert channel**: Pushover
  - **Current normal behavior**: if free space is above 200 GB, workflow does nothing.
  - **Notes**: n8n must remain local-only. `N8N_SECURE_COOKIE=false` is intentionally set for LAN HTTP access and n8n must not be exposed publicly with this setting. n8n `.env` allows only fs builtin for this workflow: `NODE_FUNCTION_ALLOW_BUILTIN=fs`. `PUSHOVER_APP_TOKEN` and `PUSHOVER_USER_KEY` are stored in `.env` and must not be printed or documented as a secret value.

- **Workflow name**: Homelab Status API
  - **Status**: active. Production-only GET endpoint: `http://192.168.10.12:5678/webhook/homelab-status`.
  - **Purpose**: read-only, bounded status response for the separate iPhone Scriptable widgets. It returns schema version, timestamp, approved Fedora CPU/RAM/uptime/local and NAS capacity/newest-backup summary, and approved Mac mini CPU/RAM/internal and SSD-MINI capacity/uptime/Ollama/newest-backup summary. Values remain bytes and seconds; no raw upstream data is returned.
  - **Sources**: internal `http://glances:61208/api/4/{cpu,mem,uptime}`; read-only `/host/home/patryk`; a read-only bind of `/mnt/nas-minipc-backup`; the strict local/NAS `homelab_backup_YYYY-MM-DD_HH-MM.tar.gz` inventory; and fixed Mac source `http://192.168.10.13:3005/status` with five-second timeout, schema version `1`, host `macmini`, and approved-field allowlist.
  - **NAS safety**: `/home/patryk/scripts/nas-mount-check.sh` writes `/home/patryk/scripts/nas-mount-status.json` only after confirming the expected CIFS mount and its read/write health; it removes the marker on failure. The API uses a marker no older than ten minutes and otherwise returns NAS fields as unavailable/null without treating the automount directory as NAS storage.
  - **Partial failure**: a failed Mac source produces only a null/unavailable Mac section and concise error indicator; Fedora data remains available. Unknown values are null, never fabricated zeroes.
  - **Security**: firewalld accepts port `5678` only from `192.168.10.0/24` and Tailscale IPv4/IPv6 ranges before rejecting other sources; the Webhook node enforces the same IP allowlist. Header Auth is intentionally not configured until its credential is created manually in n8n; no secret is stored in the workflow or documented here.
  - **Side effects and retention**: uses no Execute Command node and cannot trigger commands, backups, updates, SMART tests, notifications, or Ollama inference. Successful and error execution payload retention is disabled for this high-frequency endpoint. Existing n8n data and Compose backup coverage includes its workflow state and read-only mount configuration.
  - **Safe validation**: `curl -fsS --connect-timeout 3 --max-time 12 http://192.168.10.12:5678/webhook/homelab-status | jq .` (do not add a token to the URL).

## iPhone Scriptable widgets

The final iPhone setup uses two separate medium-sized Scriptable widgets and two separate scripts. Both fetch the same production JSON endpoint; they do not create, select, or require separate n8n workflows.

- **Production endpoint**: `http://192.168.10.12:5678/webhook/homelab-status`
- **Script names**: `MiniPC Status` and `Mac Mini Status`.
- Configure each iPhone widget to use its corresponding Scriptable script. The old combined MiniPC/Mac mini widget is obsolete and is not part of the current setup.
- No Header Auth token is embedded in either script. Access remains through trusted LAN or Tailscale routing and the existing endpoint/firewall/Webhook IP restrictions. Do not add or document a secret.

### MiniPC Status widget

- All labels are English; title is `MiniPC`.
- Shows rounded CPU and RAM percentages, then `Disk:` and `NAS:` with only their current free capacity, `Uptime:`, `Last Backup:`, `Backup:`, and footer `Updated:` using the source timestamp.
- `Last Backup:` is directly above `Backup:` and uses compact relative age, for example `12m ago`, `3h ago`, or `2d 4h ago`.
- `Backup:` maps status to normalized English: `OK`, `Local only`, `NAS unavailable`, `Missing`, or `Unknown`.

### Mac Mini Status widget

- All labels are English; title is `Mac mini`.
- Shows rounded CPU and RAM percentages, then `Internal SSD:` and `SSD-MINI:` with only their current free capacity, `Uptime:`, `Local Model:`, `Backup:`, and footer `Updated:` using the Mac mini source timestamp.
- `Local Model:` shows the loaded Ollama model name or names when present; when no model is loaded it displays exactly `No`. It never displays `unloaded`, `without model`, or `Yes`.
- `Backup:` uses the same normalized English status values as the MiniPC widget.

### Common presentation and refresh rules

- Storage values contain only the rounded free capacity and unit after the label, such as `Disk: 927 GB`; do not append `free` or `wolne`.
- Use rounded GB below 1 TB and one decimal place in TB at or above 1 TB. CPU and RAM use rounded percentages. Uptime uses compact English, such as `2d 5h` or `8h`. Unavailable values display an em dash.
- The green/red indicator in the widget title represents online state.
- Both scripts set exactly:

  ```javascript
  widget.refreshAfterDate = new Date(Date.now() + 10 * 60 * 1000);
  ```

  This requests the next refresh no earlier than approximately ten minutes; iOS controls the actual schedule and may refresh later. A failed request schedules a retry request after approximately five minutes. Manually running either Scriptable script fetches fresh data immediately.

**Planned n8n workflows still pending**:

- No duplicate n8n Paperless watchdog is planned right now. Paperless consistency checking already exists as a systemd timer and should not be duplicated in n8n without a clear new use case.


## Systemd Workflows / Timers

- **Workflow name**: RAM 24-hour sampler
  - **Schedule**: every 5 minutes from user-manager startup (`OnBootSec=2min`, `OnUnitActiveSec=5min`, `Persistent=true`).
  - **Implementation**: `/home/patryk/scripts/ram-sample.sh`, `/home/patryk/.config/systemd/user/ram-sample.service`, and `/home/patryk/.config/systemd/user/ram-sample.timer`.
  - **Data**: appends epoch timestamps and live RAM-used percentages to `/home/patryk/scripts/ram-24h.log`, pruning entries older than 24 hours on every run. The log is mode 600, bounded, contains no secrets, and is transient operational history rather than recovery data.
  - **Purpose and safety**: supplies the daily report's `24h high` value. It only reads current memory counters and writes its small rolling log; it installs nothing and has no new dependencies.

- **Workflow name**: Wildcard TLS certificate expiry checker
  - **Schedule**: weekly on Monday at 08:00 through a systemd user timer (`Persistent=true`, up to 10 minutes accuracy delay). This is clear of the backup, DIUN, title-sync, report, SMART, and Paperless slots.
  - **Implementation**: `/home/patryk/scripts/tls-cert-expiry-check.sh`, `/home/patryk/.config/systemd/user/tls-cert-expiry-check.service`, and `/home/patryk/.config/systemd/user/tls-cert-expiry-check.timer`.
  - **Live check**: connects directly to `192.168.10.12:443` with OpenSSL and SNI `kuma.patrykw.uk`, then reads the served wildcard `*.patrykw.uk` certificate's `notAfter` value. It does not inspect Caddy storage and does not require hostname DNS resolution.
  - **Alerting**: silent above 30 days remaining; sends a weekly Pushover reminder with the exact UTC expiry and days remaining at 30 days or less; sends `MiniPC - Certificate Expiry - EXPIRED` every run after expiry; and sends a distinct high-priority `MiniPC - Certificate Expiry` failure alert if connection, handshake, certificate extraction, or date parsing fails.
  - **Safety and manual checks**: strictly read-only; it never renews certificates or restarts/touches Caddy. `/home/patryk/scripts/tls-cert-expiry-check.sh --dry-run` performs the live query without Pushover. `/home/patryk/scripts/tls-cert-expiry-check.sh --test-notification` sends only a clearly labelled format test.

- **Workflow name**: SMART health checker
  - **Schedule**: daily at 11:00, `Persistent=true`, with up to 10 minutes systemd accuracy delay. This is intentionally separate from the 03:30 backup and 09:00–09:30 report timers.
  - **Implementation**: `/home/patryk/scripts/smart-health-check.sh`, `/etc/systemd/system/smart-health-check.service`, and `/etc/systemd/system/smart-health-check.timer`.
  - **Package**: `smartmontools-7.5-6.fc44.x86_64`.
  - **Monitored device**: only the internal non-removable SATA SSD `/dev/sda`, FuturePath M600 SATA 2242 1TB, serial `250117AV1T000359`. The Synology CIFS mount is remote storage and is not probed with `smartctl`; zram, partitions, Docker/device-mapper storage, network filesystems, and removable devices are excluded.
  - **Useful health data**: ATA SMART overall health, temperature, reallocated-sector and reallocated-event counts, pending and offline-uncorrectable sectors, program/erase failure counts, UDMA CRC errors, and ATA error-log presence. Baseline on 2026-07-24: overall health PASSED, 45 C, and all monitored error counters/logs 0. The SSD reports 100 available reserved space and 0 percentage-used endurance indicator through its ATA statistics log.
  - **Limitations**: the model is not in the Smartmontools 7.5 drive database, so several vendor attributes (160, 161, 163–169, 245) have no reliable meaning and are not used for alerting. Extended `smartctl -x` exits with a command-error bit because some optional data is unsupported; routine monitoring deliberately uses the validated read-only `smartctl -H -A -l error` query instead.
  - **Alerting**: every SMART-monitor Pushover notification uses the fixed title `MiniPC - Disk Alert`. One alert is sent for a newly failed overall health status, new/increased severe error counters, a non-empty ATA error log, or temperature at/above 70 C. The 70 C threshold is conservative relative to the device-reported 100 C maximum. The checker alerts after two consecutive unreadable/missing-device results, not one transient failure. It does not send daily success messages or repeat unchanged permanent media errors. It sends recovery only when SMART readability returns or an alerted temperature falls below threshold; permanent counters are never described as recovered.
  - **Manual checks**: `sudo /usr/bin/bash /home/patryk/scripts/smart-health-check.sh --dry-run` performs a real read-only check without Pushover or state writes. `sudo /usr/bin/bash /home/patryk/scripts/smart-health-check.sh --test-notification` sends an explicitly labelled notification-format test only; it does not claim disk failure or change disk state.
  - **Self-tests**: none are scheduled. Any SMART self-test policy remains an explicit later maintenance decision.

- **Workflow name**: Trivy running-image vulnerability scanner
  - **Schedule**: daily at 12:10 through `trivy-image-scan.timer` (`Persistent=true`, one-minute accuracy). This is clear of the 11:00 SMART check, 13:20 public-documentation push, and all other established timer slots.
  - **Package and source**: `trivy-0.72.0-1` from Aqua Security's official DNF RPM repository, `https://aquasecurity.github.io/trivy-repo/rpm/releases/$basearch/`, using the post-incident RPM signing key installed as `/etc/pki/rpm-gpg/RPM-GPG-KEY-trivy-2026` (fingerprint `825A D903 6F7C 850E 6A6F ED49 35B8 ACA4 4FD9 CA9F`, created 2026-04-16). This is newer than the compromised v0.69.4 release.
  - **Implementation**: `/home/patryk/scripts/trivy-image-scan.sh` (mode 700), `/etc/systemd/system/trivy-image-scan.service`, and `/etc/systemd/system/trivy-image-scan.timer`; the system service runs the CLI as `patryk` with Docker-group access.
  - **Dashboard**: Trivy is CLI-only and has no web UI or health endpoint, so no Homarr application tile is expected.
  - **Scope and safety**: deduplicates only the images reported by `docker ps --format '{{.Image}}'`, then performs vulnerability-only `trivy image` scans at HIGH/CRITICAL severity in JSON mode with exit code 0. It never scans unused pulled images, exposes no port or UI, adds no Docker socket mount or long-lived scanner container, and never fixes, updates, restarts, or otherwise changes an image or container.
  - **Alerting and state**: compares each running image's HIGH/CRITICAL CVE identifiers to `/home/patryk/scripts/trivy-image-scan-state/baseline.json`; it uses the fixed Pushover title `MiniPC - Trivy Alert` only for newly appearing findings (including a newly in-scope image), never for an unchanged result or routine all-clear. The first successful run initializes the baseline silently.
  - **Recovery state decision**: the live baseline and lock under `/home/patryk/scripts/trivy-image-scan-state/` are intentionally excluded from backups and are reset fresh after recovery. They are transient alert-suppression state; restoring a stale baseline could hide a new finding, while a fresh baseline is safer and produces no initialization alert.

- **Workflow name**: Paperless consistency checker
  - **Schedule**: daily 19:00
  - **Purpose**: alert if Paperless documents have missing/wrong Document Type, wrong Storage Path, or UI Title not matching DD-MM-YYYY - 000000x. Paperless consistency automation now safely auto-fills missing Document Type based on Storage Path before checking.
  - **Checks**: Read-only checks except for the explicit safe autofill step via `docker compose exec` into the `paperless` container running a Django script. It never overwrites an existing Document Type, does not guess from OCR, and alerts if Storage Path is missing/unknown.
  - **Mapping**: Storage Path CMR -> Document Type CMR by default; customs documents use Document Type Dokument celny with the same CMR Storage Path; Storage Path FAKTURY -> Document Type Faktura.
  - **Alert channel**: Pushover
  - **Implementation**: Systemd user timer (`paperless-consistency-check.timer`) and scripts.
  - **Script paths**:
    `/home/patryk/scripts/paperless-autofill-document-type.sh`
    `/home/patryk/scripts/paperless-consistency-check.sh`
    `/home/patryk/scripts/paperless-consistency-check-notify.sh`

- **Workflow name**: Homelab repo collector
  - **Schedule**: daily at 09:00, `Persistent=true`.
  - **Purpose**: copies an explicit allowlist of homelab documentation, Compose files, operational scripts, and systemd unit files into the local Git repository `/home/patryk/homelab-repo`, giving configuration changes history and a rollback path. This is a one-way, read-only snapshot; nothing is deployed from `/home/patryk/homelab-repo`. The snapshot is pushed to Forgejo over SSH via a dedicated deploy key.
  - **Implementation**: `/home/patryk/scripts/collect-homelab-repo.sh`, `/home/patryk/.config/systemd/user/homelab-repo-collect.service`, and `/home/patryk/.config/systemd/user/homelab-repo-collect.timer`.
  - **Allowlist**: `HOMELAB.md`; each service's `compose.yml`/`docker-compose.yml` plus `reverse-proxy/Caddyfile` and the retained `homelab-docs/server.py`; every `*.sh` in `/home/patryk/scripts`; the homelab-related systemd user and system unit files. Whole directories and wildcards over unknown files are never copied.
  - **Excluded**: all `.env` files; live service data (databases, volumes). `compose/paperless/docker-compose.yml` is now included on the allowlist (Forgejo commit `2a845a1`), since it no longer contains a hardcoded secret.
  - **Secret scan gate**: before staging, every candidate file is scanned for keyword-plus-assignment patterns (`token`/`api_key`/`password`/`secret`/`PUSHOVER` followed by `:`/`=` and a literal, non-variable value), the Uptime Kuma `/api/push/` path literal, and long mixed-case alphanumeric strings (high-entropy heuristic). Any match aborts the whole run before anything is committed, printing only the offending file path and line number, never the matched value.
  - **Idempotency**: `git add -A` plus a commit only when the tree actually changed; the collector's own source is excluded from its own scan since it contains the pattern definitions as literal text.

- **Workflow name**: Public GitHub docs push (`homelab-docs-public`)
  - **Schedule**: event-driven on every change to `/home/patryk/docker/HOMELAB.md` via `homelab-docs-public-github-push.path`, plus a daily fallback at 13:20 (`Persistent=true`). 13:20 is deliberately clear of the 03:30 backup, 04:00 DIUN, 06:30/18:30 title-sync, Monday 08:00 TLS check, 09:00/09:30 notifier/report slots, 11:00 SMART, and 19:00 Paperless slots.
  - **Purpose**: one-way mirror of **`HOMELAB.md` only** to the public GitHub repository `https://github.com/patryk-homelab/homelab-docs-public`, so AI chat tools can fetch the canonical documentation from a stable raw-content URL: `https://raw.githubusercontent.com/patryk-homelab/homelab-docs-public/main/HOMELAB.md`. No authentication is needed to read it; a plain `curl` returns the current content.
  - **Scope**: nothing other than `HOMELAB.md` is ever copied, committed, or pushed, and nothing is ever deployed from the working copy back to the canonical file. This is **separate from and unrelated to** the `homelab-repo` -> Forgejo snapshot pipeline, which keeps its own script, repository, units, and deploy key.
  - **Implementation**: `/home/patryk/scripts/push-homelab-docs-public-github.sh` (mode 700), working copy `/home/patryk/homelab-docs-public-github`, and systemd user units `homelab-docs-public-github-push.service`, `.path`, and `.timer`.
  - **Transport**: SSH via the dedicated `ssh_config` alias `github-homelab-docs-public` (`HostName github.com`, `IdentitiesOnly yes`), a dedicated ed25519 repository-scoped deploy key with write access at `/home/patryk/.ssh/github-homelab-docs-public-deploy` (mode 600, private key never printed/logged), and a pinned `UserKnownHostsFile` populated from GitHub's published ed25519 host-key fingerprint. The alias is used in the git remote instead of raw `github.com` so no other key or host config can satisfy this push.
  - **Secret scan gate**: before staging, the copied `HOMELAB.md` is scanned with the same style of check the `homelab-repo` collector uses — keyword-plus-assignment patterns (`token`/`api_key`/`password`/`secret`/`PUSHOVER` followed by `:`/`=` and a literal, non-variable value), the Uptime Kuma `/api/push/` path literal followed by ID characters, and long mixed-case alphanumeric strings (high-entropy heuristic). Any match aborts the run before anything is committed or pushed, printing only the line number, never the matched value. Because the target repository is public, this gate is the load-bearing control; the file's existing no-secrets policy remains the primary one.
  - **Idempotency**: commit only when `HOMELAB.md` actually changed; an unchanged run exits cleanly with no network call.
  - **Shared repository, two independent writers**: the GitHub repository holds **both hosts' documentation** — `HOMELAB.md` pushed from the Fedora miniPC by this script, and `HOMELAB-MACMINI.md` pushed independently by the Mac mini from its own script. Neither script stages anything but its own file (`git add HOMELAB.md`, never `git add -A`), so the two writers operate on disjoint paths and cannot overwrite or delete each other's document. Raw URLs: `https://raw.githubusercontent.com/patryk-homelab/homelab-docs-public/main/HOMELAB.md` and `.../HOMELAB-MACMINI.md`.
  - **Secret scanning is per-host, by design**: each host's own gate scans its own file before that file ever reaches the shared repository — this script gates `HOMELAB.md`, the Mac mini's script gates `HOMELAB-MACMINI.md`. No shared or cross-host gate is needed or wanted: neither script can introduce content into the other's file, so each gate fully covers everything its host publishes.
  - **Race handling**: because either host can advance `origin/main` between this host's commit and its push, a plain push can be rejected as a non-fast-forward (observed on 2026-07-26, when a Mac mini push landed between two runs here and required manual reconciliation). The script now runs `git fetch` plus `git rebase origin/main` before each push, retrying up to **3** times with a 3s/6s backoff, and resolves such races automatically with no manual step. It **never force-pushes**. If all attempts fail, or a rebase conflicts, it aborts the rebase cleanly and exits non-zero, leaving the local commit intact to be carried along by the next successful run.
  - **Separate from the decommissioned `homelab-docs-public` container**: the retained Docker service is intentionally stopped. This workflow instead publishes a Git-committed snapshot on GitHub; the two share a name and purpose but no code, and the GitHub copy is only as fresh as the last successful push.

- **Workflow name**: Mac mini iCloud Drive documentation delivery
  - **Schedule**: event-driven on every change to `/home/patryk/docker/HOMELAB.md` via `homelab-macmini-icloud-push.path`, plus a `Persistent=true` backstop two minutes after user-manager startup and every 15 minutes thereafter. The shorter cadence than the daily GitHub fallback provides a safety margin for the always-on Mac mini while unchanged runs make no network call.
  - **Purpose and scope**: one-way delivery of **`HOMELAB.md` only** from this Fedora host to `/Users/patrykmac/Library/Mobile Documents/com~apple~CloudDocs/DRIVE/HOMELAB.md` on the Mac mini. It is additional to and independent of the public GitHub mirror push; GitHub and Forgejo configuration are unchanged.
  - **Implementation**: `/home/patryk/scripts/push-homelab-macmini-icloud.sh` (mode 700), with systemd user units `homelab-macmini-icloud-push.service`, `.path`, and `.timer`.
  - **Transport and restriction**: SSH to `patrykmac@192.168.10.13` using only `/home/patryk/.ssh/macmini-icloud-push-deploy` (mode 600), `IdentitiesOnly=yes`, and the pinned Mac host key in `/home/patryk/.ssh/known_hosts`. The Mac mini authorizes this key with a forced command that receives this stream only; it grants no shell and does not permit Fedora to name another destination. The private key is never printed or logged.
  - **Idempotency and failure handling**: the script compares the SHA-256 of the source with the hash from the last successful delivery. Matching content exits without connecting; a changed source is sent up to 3 times with 3s/6s backoff. The state hash is updated only after a successful receiver exit status, so a failed transfer is retried later.

## Infrastructure services

| Service | Purpose | Type | Important path/command | Notes |
|---|---|---|---|---|
| Docker | Container runtime for homelab apps | systemd/system | `systemctl status docker`, `docker ps` | `docker.service` is active. |
| Cudy WR3600 router | LAN gateway, DHCP, OpenVPN fallback access | external router | `http://192.168.10.1` | Router must stay reachable independently from miniPC services. DHCP DNS currently uses primary `192.168.10.12` (Fedora miniPC AdGuard Home) and secondary `192.168.10.13` (Mac mini AdGuard Home), and both should stay mirrored because clients may use either server. |
| Cudy OpenVPN Server | Emergency home LAN access | external router VPN | Router panel `http://192.168.10.1` over VPN | Critical fallback for router management if DNS, miniPC, Tailscale, or AdGuard Home are unavailable. |
| Tailscale | Private remote access to miniPC and services | systemd/system | `tailscale status`, `systemctl status tailscaled` | miniPC Tailscale IP is `100.118.164.107`. |
| SSH | Remote shell access | system/system | `ss -tulpn`, port `22` | Listening on IPv4 and IPv6. |
| Avahi / mDNS | Local `.local` hostname | systemd/system | `systemctl status avahi-daemon`, `avahi-resolve -n fedora.local` | Avahi is active, advertising `fedora.local`. SSH target is `ssh patryk@fedora.local`. Configured to allow only `enp1s0` interface, use IPv4 only (IPv6 publishing disabled), and static hostname is set to `fedora`. |
| Samba / SMB | Authenticated file access from iPhone Files, Samsung Fold, and Mac | systemd/system | `systemctl status smb`, `/etc/samba/smb.conf` | Bound only to `lo` and `enp1s0`. Guest access disabled. Final working shares expose existing folders under `/home/patryk`: `documents`, `backups`, and `downloads`. Administrative folders like `/home/patryk/docker` and `/home/patryk/scripts` are intentionally not exposed over SMB. The physical folder for `documents` remains `/home/patryk/Documents`. |
| GNOME Remote Desktop / RDP | Remote desktop access | systemd/system | `systemctl status gnome-remote-desktop`, port `3389` | Listening on port `3389`. |
| SMB NAS mount | Backup destination on Synology NAS | system/manual mount | `mountpoint /mnt/nas-minipc-backup`, `df -h /mnt/nas-minipc-backup` | Mounted from `//192.168.10.92/minipc-backup`. |
| NAS Wake-on-LAN recovery | Can wake miniPC from LAN | NAS feature | miniPC MAC `68:1d:ef:4e:2e:a9` | Use after connecting through OpenVPN when miniPC is powered off but router and NAS are available. |
| NAS mount alert | Verifies NAS backup mount health | systemd/timer + script | `/home/patryk/scripts/nas-mount-check.sh`, `systemctl status nas-mount-check.timer` | Runs every 3 minutes, checks `/mnt/nas-minipc-backup`, writes/reads/deletes a test file, reports to Uptime Kuma Push monitor set to 300 seconds / 5 minutes, and Uptime Kuma sends Pushover notification. This avoids false warning/flapping caused by exact 5-minute scheduling drift. |
| Homelab backup script | Creates local archive and copies it to NAS | manual/script | `/home/patryk/scripts/homelab-backup.sh` | Requires the expected `//192.168.10.92/minipc-backup` CIFS mount before archive creation or NAS copy; supports safe `--preflight`; sends Uptime Kuma push after success without printing the push URL. |
| Homelab restore verification script | Safely validates backup archives without restoring live files | manual/script | `/home/patryk/scripts/homelab-restore-verify.sh` | Checks tar integrity, extracts only to `/tmp`, verifies important recovery paths, and prints a PASS/FAIL summary. |
| Homelab backup systemd timer | Runs the homelab backup daily | systemd/timer | `systemctl status homelab-backup.timer` | Enabled. Next run is scheduled by systemd with randomized delay. |
| Homelab repo | Local Git repository versioning homelab docs, Compose files, scripts, and systemd units | Git repository | `/home/patryk/homelab-repo` | One-way, read-only snapshot populated by an explicit allowlist collector; not a deployment source. The snapshot is pushed to Forgejo over SSH via a dedicated deploy key. See Systemd Workflows / Timers and Backup coverage. |
| Homelab repo collector | Populates `/home/patryk/homelab-repo` from an explicit allowlist | systemd/user timer + script | `/home/patryk/scripts/collect-homelab-repo.sh`, `systemctl --user status homelab-repo-collect.timer` | Runs daily at 09:00. Includes a secret-scan gate that aborts the run before committing if any staged file matches a secret-like pattern. |
| Daily homelab report | Sends compact miniPC status report through Pushover | systemd/timer + script | `/home/patryk/scripts/daily-homelab-report.sh`, `systemctl status daily-homelab-report.timer` | Runs daily at 09:30. Body order is Uptime, RAM current/24-hour high, NAS, Backup, Timers, Reboot Needed, Docker counts/problems, then Updates and Recommendations; there is no Host, Kernel, or Load line. Updates uses a header plus compact Fedora count/security-critical and DIUN count/critical lines. Individual critical/important package or image names appear only on the final Recommendations line; otherwise that line explicitly says none need review. The former 09:15 standalone update notifier is retired. |
| RAM 24-hour sampler | Supplies rolling memory high-water data to the daily report | systemd/user timer + script | `/home/patryk/scripts/ram-sample.sh`, `systemctl --user status ram-sample.timer` | Samples live RAM-used percentage every 5 minutes and prunes `/home/patryk/scripts/ram-24h.log` to the latest 24 hours. |
| TLS certificate expiry check | Monitors the live DNS-01 Caddy wildcard certificate | systemd/user timer + script | `/home/patryk/scripts/tls-cert-expiry-check.sh`, `systemctl --user status tls-cert-expiry-check.timer` | Runs Monday at 08:00. Silent above 30 days, reminds weekly in the final 30 days, alerts every run after expiry, and reports endpoint/check failures separately. Read-only: no renewal or Caddy action. |
| DIUN | Docker image update checks | Docker Compose | `/home/patryk/docker/diun` | Installed and active. Runs daily at 04:00. Used as the Docker image update source for the Daily homelab report. Does not auto-update containers. Uses Docker socket read-only. Avoid adding duplicate standalone Docker update notifications unless explicitly requested. |
| Trivy image vulnerability scanner | Detects new HIGH/CRITICAL CVEs in images used by running containers | systemd/timer + RPM CLI | `/home/patryk/scripts/trivy-image-scan.sh`, `systemctl status trivy-image-scan.timer` | Trivy `0.72.0-1` from Aqua's official RPM repository. Runs daily at 12:10; scans only deduplicated `docker ps` images, never fixes or changes them, and alerts through `MiniPC - Trivy Alert` only when the per-image baseline detects new CVE identifiers. No port, web UI, Docker socket mount, or long-lived scanner container. |
| Anti-sleep host configuration | Prevents the miniPC from sleeping/suspending | systemd/GDM config + timer | `systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target`, `/home/patryk/scripts/anti-sleep-check.sh`, `systemctl status anti-sleep-check.timer` | Sleep is disabled through masked systemd sleep targets, logind and sleep config drop-ins, and GDM power settings. `anti-sleep-check.timer` verifies this every 15 minutes and sends Pushover notification only for new suspicious suspend attempts after baseline or config drift. |
| Forgejo | Self-hosted Git server backing `homelab-repo`'s one-way push | Docker Compose | `/home/patryk/docker/forgejo/compose.yml`, `http://192.168.10.12:3300`, SSH `2222` | Data at `/home/patryk/docker/forgejo/data` (SQLite database, repos, `app.ini`, and host SSH keys under a root-owned `ssh/` subdirectory). Deploy key used for the `homelab-repo` push: `/home/patryk/.ssh/forgejo-homelab-repo-deploy` (mode 600, private key never printed/logged). Reachable via both LAN and Tailscale by default, per the FedoraWorkstation zone's stock behavior; not further restricted. |
| Public GitHub docs push | One-way mirror of `HOMELAB.md` only to the public GitHub repo `homelab-docs-public` | systemd/user path + timer + script | `/home/patryk/scripts/push-homelab-docs-public-github.sh`, `systemctl --user status homelab-docs-public-github-push.path` | Target repo `https://github.com/patryk-homelab/homelab-docs-public`; raw URL `https://raw.githubusercontent.com/patryk-homelab/homelab-docs-public/main/HOMELAB.md` (public, no auth). Triggered by the same `HOMELAB.md` change event as the mirror sync, plus a 13:20 daily fallback. Working copy `/home/patryk/homelab-docs-public-github`. Pushes over SSH as alias `github-homelab-docs-public` with the dedicated deploy key `/home/patryk/.ssh/github-homelab-docs-public-deploy` (mode 600, never printed). Secret-scan gate aborts before any commit or push. The repository is shared with the Mac mini, which pushes `HOMELAB-MACMINI.md` from its own script; each host gates and pushes only its own file, and this script auto-resolves non-fast-forward races with `fetch` + `rebase` + bounded retry, never a force push. Separate from the `homelab-repo`/Forgejo snapshot and from the decommissioned `homelab-docs-public` container. See Systemd Workflows / Timers and Backup coverage. |
| Mac mini iCloud documentation delivery | One-way delivery of `HOMELAB.md` only to the Mac mini's iCloud Drive | systemd/user path + timer + restricted SSH receiver | `/home/patryk/scripts/push-homelab-macmini-icloud.sh`, `systemctl --user status homelab-macmini-icloud-push.path` | Target `/Users/patrykmac/Library/Mobile Documents/com~apple~CloudDocs/DRIVE/HOMELAB.md` on `patrykmac@192.168.10.13`; the Mac-side forced command accepts this stream only and grants no shell. Triggered by the same `HOMELAB.md` change event as the SMB and GitHub mirrors, plus a 15-minute persistent fallback. Uses its own dedicated key `/home/patryk/.ssh/macmini-icloud-push-deploy` (mode 600; never printed). Hash state advances only after success, and changed content retries 3 times with 3s/6s backoff. Additional to and independent of the GitHub mirror; GitHub/Forgejo configuration is unchanged. |
| Codex CLI | Local coding/ops assistant used to maintain documentation and configs | manual/tool | `codex` | Do not store secrets in prompts, logs, or docs. |

## Anti-sleep monitoring

- miniPC is configured for 24/7 operation and should not suspend, sleep, hibernate, or hybrid-sleep automatically.
- `sleep.target`, `suspend.target`, `hibernate.target`, and `hybrid-sleep.target` are masked.
- Before Fedora/system updates, verify that `sleep.target`, `suspend.target`, `hibernate.target`, and `hybrid-sleep.target` are still masked.
- After Fedora/system or kernel updates and after any reboot, verify the sleep/suspend/hibernate masks again. Do not assume they survived the update until checked.
- GDM inactive sleep is disabled through GDM power settings.
- logind config disables suspend-related actions:
  - `/etc/systemd/logind.conf.d/99-disable-suspend.conf`
  - expected values include `HandleSuspendKey=ignore`, `HandleHibernateKey=ignore`, `HandleLidSwitch=ignore`, `HandleLidSwitchExternalPower=ignore`, `IdleAction=ignore`, and `IdleActionSec=0`.
- systemd sleep config disables suspend and hibernation:
  - `/etc/systemd/sleep.conf.d/99-disable-all-sleep.conf`
  - expected values include `AllowSuspend=no`, `AllowHibernation=no`, `AllowHybridSleep=no`, and `AllowSuspendThenHibernate=no`.
- Monitoring script:
  - `/home/patryk/scripts/anti-sleep-check.sh`
- Baseline file:
  - `/home/patryk/scripts/anti-sleep-check.baseline`
- Last diagnostic log:
  - `/home/patryk/scripts/anti-sleep-check.last.log`
- systemd units:
  - `/etc/systemd/system/anti-sleep-check.service`
  - `/etc/systemd/system/anti-sleep-check.timer`
- Timer interval:
  - `OnBootSec=5min`
  - `OnUnitActiveSec=15min`
  - `AccuracySec=1min`
  - `Persistent=true`
- Alerting:
  - Pushover alert is sent only for new suspicious suspend/sleep/hibernate entries after the baseline timestamp or for anti-sleep config drift.
  - The script still logs suspicious entries from the last 72h for visibility, but does not alert on historical entries before baseline.

## Notifications

- Default homelab notifications use **Pushover**. Uptime Kuma uses Pushover.
- Central generic helper: `/home/patryk/scripts/notify.sh`
- Pushover specific helper: `/home/patryk/scripts/notify-pushover.sh`
- Pushover secrets: `/home/patryk/scripts/pushover.env` and `/home/patryk/docker/n8n/.env` (for n8n). Secret values are strictly not documented here.
- Pushover is the active/default notification provider for the homelab. changedetection.io uses Pushover through Apprise with `pover://...` configured manually in its UI. Do not document or expose the actual Pushover tokens or keys.
- Used for script notifications that do not need Uptime Kuma monitoring, such as reboot-needed, daily report, SMART, and certificate-expiry notices.
- Daily homelab report notifications use the title `MiniPC - Daily Report`.
  - The title gets ` - WARN` only for a failed, missing, mismatched, or more-than-30-hours-old backup; an unavailable or unhealthy `/mnt/nas-minipc-backup` mount; a sampled rolling 24-hour RAM high over 90%; or a missing/inactive important timer.
  - Available Fedora or DIUN updates, Reboot Needed, and Docker non-running container counts remain in the report body but do not add ` - WARN` by themselves.
- Example usage:

```bash
/home/patryk/scripts/notify.sh "MiniPC test" "Message"
```

- Reboot Needed notifier: `/home/patryk/scripts/reboot-needed-check.sh`
- Uses the shared helper: `/home/patryk/scripts/notify.sh`
- Runs daily at 09:00 through `reboot-needed-check.timer` when the systemd unit and timer are installed.
- Sends notification only when Fedora likely needs a reboot, such as after a newer installed kernel differs from the running kernel or `/run/reboot-required` exists.
- Does not use Uptime Kuma.
- Does not install updates and does not reboot automatically.
- Avoids repeated spam by storing the last notified state in `/home/patryk/scripts/reboot-needed-check.state`.
- The former standalone Update Available notifier, its 09:15 timer/service, script, and state file are retired. Its read-only `dnf5`/`dnf` availability check now supplies the daily report's compact `Fedora: <count> available (security/critical: yes|no)` line.
- Boot notification: `/home/patryk/scripts/boot-notify.sh`
- Uses the shared helper: `/home/patryk/scripts/notify.sh`
- Sends notification every time the miniPC boots through `boot-notify.service` when the systemd service is installed and enabled.
- Does not use Uptime Kuma.
- Includes hostname, boot time, running kernel, LAN IP if available, and Tailscale IP if available.
- Retries notification delivery because DNS/network may not be ready immediately after boot.
- Daily Homelab Report: `/home/patryk/scripts/daily-homelab-report.sh`
- Uses the shared helper: `/home/patryk/scripts/notify.sh`
- Runs daily at 09:30 through `daily-homelab-report.timer` when the systemd unit and timer are installed.
- Sends one compact report in this order: Uptime; current RAM plus sampled 24-hour high; NAS mount health; backup status/latest age; important-timer health; Reboot Needed; Docker running/total counts and non-running problems; the three-line Updates section (`Updates:`, compact Fedora status, compact DIUN status); and a final Recommendations line. Host, Kernel, and Load are omitted. The DIUN status never lists image names; the Recommendations line names only services/images classified as critical/important by the existing checks, or explicitly states that none need review.
- Appends ` - WARN` to the title only for a failed/missing/mismatched/stale backup, unhealthy or unavailable NAS mount, rolling 24-hour RAM high over 90%, or missing/inactive important timer. Fedora/DIUN updates, Reboot Needed, and Docker non-running counts do not trigger the suffix by themselves.
- Does not use Uptime Kuma as a monitor.
- Does not install updates, reboot, restart containers, or run speedtests. Updates remain manual/planned maintenance only.
- `/home/patryk/scripts/daily-homelab-report.sh --dry-run` performs the live read-only report checks and prints the exact would-be report without Pushover. `--test-notification` sends only a clearly labelled notification-format test.

## Docker compose folders

Current expected Docker status: `33/33 running` (includes Docker Compose-managed containers and the plain-Docker `uptime-kuma` container).

Known Compose files under `/home/patryk/docker`:

- `/home/patryk/docker/adguardhome/compose.yml`
- `/home/patryk/docker/freshrss/compose.yml`
- `/home/patryk/docker/homarr/compose.yml`
- `/home/patryk/docker/glances/compose.yml`
- `/home/patryk/docker/snapotter/compose.yml`
- `/home/patryk/docker/diun/compose.yml`
- `/home/patryk/docker/forgejo/compose.yml`
- `/home/patryk/docker/speedtest-tracker/compose.yml`
- `/home/patryk/docker/changedetection/compose.yml`
- `/home/patryk/docker/wallos/compose.yml`
- `/home/patryk/docker/netalertx/compose.yml`
- `/home/patryk/docker/cyberchef/docker-compose.yml`
- `/home/patryk/docker/stirling-pdf/compose.yml`
- `/home/patryk/docker/watcharr/compose.yml`
- `/home/patryk/docker/czkawka/compose.yml`
- `/home/patryk/docker/filebrowser/compose.yml`
- `/home/patryk/docker/rss-bridge/compose.yml`
- `/home/patryk/docker/n8n/docker-compose.yml`
- `/home/patryk/docker/upsnap/compose.yml`
- `/home/patryk/docker/reverse-proxy/compose.yml`
- `/home/patryk/docker/paperless/docker-compose.yml`
- `/home/patryk/docker/caddy/compose.yml`
- `/home/patryk/docker/shlink/compose.yml`
- `/home/patryk/docker/convertx/compose.yml`

Known service folders under `/home/patryk/docker`:

- `/home/patryk/docker/adguardhome`
- `/home/patryk/docker/freshrss`
- `/home/patryk/docker/homarr`
  - `/home/patryk/docker/homarr/data` (SQLite database, Redis state, trusted certificates, uploaded/local media, board/app/item/widget/integration and authentication data)
- `/home/patryk/docker/glances` (pinned Compose only; no persistent metrics cache)
- `/home/patryk/docker/snapotter`
  - `/home/patryk/docker/snapotter/data` (user files and local AI runtime data; backed up)
  - `/home/patryk/docker/snapotter/pgdata` (PostgreSQL 17 database; backed up)
  - `/home/patryk/docker/snapotter/redisdata` (Redis cache/queue state; intentionally excluded from recovery backup)
- `/home/patryk/docker/diun`
- `/home/patryk/docker/forgejo`
- `/home/patryk/docker/speedtest-tracker`
- `/home/patryk/docker/changedetection`
- `/home/patryk/docker/wallos`
  - `/home/patryk/docker/wallos/db` (SQLite database)
  - `/home/patryk/docker/wallos/logos` (uploaded subscription logos)
- `/home/patryk/docker/netalertx`
  - `/home/patryk/docker/netalertx/data` (application config and SQLite database)
- `/home/patryk/docker/cyberchef`
- `/home/patryk/docker/stirling-pdf`
  - `/home/patryk/docker/stirling-pdf/configs` (generated application settings and persistent state; backed up)
- `/home/patryk/docker/watcharr`
  - `/home/patryk/docker/watcharr/.env` (mode 600; generated initial admin username/password for retrieval; never injected into the container, printed, or committed)
  - `/home/patryk/docker/watcharr/data` (`watcharr.db` SQLite database, `watcharr.json` server configuration, cache, and application state; backed up during a brief consistent service stop)
- `/home/patryk/docker/czkawka`
  - `/home/patryk/docker/czkawka/config` (Czkawka persistent GUI configuration, state, and logs; `/home/patryk` is mounted in the container only as `/storage:ro`)
- `/home/patryk/docker/filebrowser`
  - `/home/patryk/docker/filebrowser/data` (FileBrowser Quantum config, BoltDB database, cache, password hashes, and encrypted TOTP secrets; private-backup-only, never part of the public documentation mirror)
- `/home/patryk/docker/rss-bridge`
  - `/home/patryk/docker/rss-bridge/config` (RSS-Bridge `config.ini.php`: enabled bridges, settings)
- `/home/patryk/docker/convertx`
  - `/home/patryk/docker/convertx/.env` (mode 600; JWT secret plus superseded historical first-run account values; never current login authority, printed, or committed)
  - `/home/patryk/docker/convertx/CREDENTIALS.md` (non-secret note defining SQLite/in-app login authority and the historical-only status of the `.env` bootstrap values)
  - `/home/patryk/docker/convertx/data` (root-owned mode 750; SQLite database, uploads, converted outputs, and job history)
- `/home/patryk/docker/n8n`
- `/home/patryk/docker/upsnap`
- `/home/patryk/docker/reverse-proxy`
- `/home/patryk/docker/paperless`
- `/home/patryk/docker/caddy` (`compose.yml`, `Caddyfile`, `Dockerfile`, `.env`; wildcard `*.patrykw.uk` DNS-01 TLS instance, 13 internal-only site blocks)
- `/home/patryk/docker/shlink`
  - `/home/patryk/docker/shlink/postgres-data` (PostgreSQL data; backed up through an online logical dump, not a blind live-file copy)
  - `/home/patryk/docker/shlink/.env` (mode 600; database password and no-role admin API key; never committed)
  - `/home/patryk/docker/shlink/Caddyfile.public` and `Dockerfile.public` (capability-free redirect-only public gateway)

The `/home/patryk/docker/homelab-docs`, `/home/patryk/docker/homelab-docs-public`, and `/home/patryk/docker/cloudflared` Compose folders, including their configs and data, remain on disk but are intentionally stopped and decommissioned for possible future reactivation.

## Backup coverage

The current backup script is:

```bash
/home/patryk/scripts/homelab-backup.sh
```

The systemd timer is:

```bash
homelab-backup.timer
```

Before creating an archive, the backup script triggers the existing automount if idle and verifies that `/mnt/nas-minipc-backup` is the expected `//192.168.10.92/minipc-backup` `cifs` filesystem. Use `/home/patryk/scripts/homelab-backup.sh --preflight` for this non-backup check.

Current backup includes:

- Docker Compose files under `/home/patryk/docker` with names `compose.yml`, `docker-compose.yml`, `.env`, `Caddyfile`, and `Dockerfile`
- Docker inventory: containers, images, and volume list
- AdGuard Home Docker volumes:
  - `adguardhome_adguard_conf`
  - `adguardhome_adguard_work`
- FreshRSS Docker volumes:
  - `freshrss_freshrss_data`
  - `freshrss_freshrss_extensions`
- Uptime Kuma Docker volume if one of the legacy volume names exists
- Homarr recovery data:
  - `/home/patryk/docker/homarr/compose.yml`
  - `/home/patryk/docker/homarr/.env` (stored without printing secret values)
  - `/home/patryk/docker/homarr/data` including SQLite, Redis state, trusted certificates, uploaded/local assets, board/app/item/widget/integration data, and authentication data
  - `/home/patryk/scripts/homarr-place-item.py`, the guarded cross-layout tRPC placement helper, archived under `scripts/`
- Glances:
  - `/home/patryk/docker/glances/compose.yml` is explicitly archived; Glances has no recovery-critical runtime metrics cache.
- Uptime Kuma bind-mounted data:
  - `/home/patryk/homelab/uptime-kuma/data`

- DIUN config and database:
  - `/home/patryk/docker/diun/diun.yml`
  - `/home/patryk/docker/diun/data`
  - archived under `bind_mounts/diun_config/`
  - archived under `bind_mounts/diun_data/`
- Speedtest Tracker config and environment:
  - `/home/patryk/docker/speedtest-tracker/config`
  - `/home/patryk/docker/speedtest-tracker/.env`
  - archived under `bind_mounts/speedtest_tracker_config/`
  - archived under `bind_mounts/speedtest_tracker_env/`
- Decommissioned Homelab Docs files (retained for possible future reactivation):
  - `/home/patryk/docker/homelab-docs/compose.yml`
  - `/home/patryk/docker/homelab-docs/server.py`
  - mounted documentation source `/home/patryk/docker/HOMELAB.md`
  - archived under `compose/homelab-docs/`, `bind_mounts/homelab_docs/`, and `docs/`
- changedetection.io:
  - `/home/patryk/docker/changedetection/compose.yml`
  - `/home/patryk/docker/changedetection/datastore`
  - notifications are configured manually in the changedetection.io UI to use Pushover through Apprise `pover://...`
  - archived under `compose/changedetection/` and `bind_mounts/changedetection_datastore/`
- Wallos:
  - `/home/patryk/docker/wallos/compose.yml`
  - `/home/patryk/docker/wallos/db` and `/home/patryk/docker/wallos/logos`
  - archived under `compose/wallos/` and `bind_mounts/wallos/wallos_data.tar.gz`
- SnapOtter:
  - `/home/patryk/docker/snapotter/compose.yml` and protected `.env` are covered by the generic Compose-file/`.env` collection without printing secret values
  - `/home/patryk/docker/snapotter/data` and `/home/patryk/docker/snapotter/pgdata` are explicitly archived under `bind_mounts/snapotter/snapotter_data.tar.gz`
  - `/home/patryk/docker/snapotter/redisdata` is Redis cache/queue state and is intentionally excluded from recovery backup
- NetAlertX:
  - `/home/patryk/docker/netalertx/compose.yml`
  - `/home/patryk/docker/netalertx/data` including application configuration and SQLite database state
  - archived under `compose/netalertx/` and `bind_mounts/netalertx_data/data.tar.gz`
- Czkawka:
  - `/home/patryk/docker/czkawka/compose.yml` (included by the generic Compose-file collection)
  - `/home/patryk/docker/czkawka/config` containing persistent application configuration, state, and logs, archived under `bind_mounts/czkawka_config/`
  - The `/home/patryk:/storage:ro` mount needs no separate Docker-backup coverage because it is only a read-only view of already-covered host user data.
  - It is intentionally not monitored by Uptime Kuma and is not part of the Daily Homelab Report.
- FileBrowser Quantum:
  - `/home/patryk/docker/filebrowser/compose.yml` and protected `.env` are covered by the generic Compose-file/`.env` collection.
  - `/home/patryk/docker/filebrowser/data` is explicitly archived under `bind_mounts/filebrowser_data/`; it contains `config.yaml`, the v1 stable BoltDB database, cache, password hashes, and encrypted TOTP secrets. This is sensitive private-backup-only material and is never added to the public GitHub documentation mirror.
- RSS-Bridge:
  - `/home/patryk/docker/rss-bridge/compose.yml` is covered by the generic Compose-file collection.
  - `/home/patryk/docker/rss-bridge/config` (the `config.ini.php` with enabled bridges and settings) is explicitly archived under `bind_mounts/rss_bridge_config/`; the image has no separate internal data volume.
- Stirling-PDF ultra-lite:
  - `/home/patryk/docker/stirling-pdf/compose.yml` is covered by the generic Compose-file collection.
  - `/home/patryk/docker/stirling-pdf/configs` is explicitly archived under `bind_mounts/stirling_pdf_configs/`; it contains generated settings and any future persistent application state.
  - There is no `.env` because this ultra-lite build has no authentication module and therefore no deployed credentials or secrets.
  - Restore verification requires both the Compose file and generated `settings.yml`.
- Watcharr:
  - `/home/patryk/docker/watcharr/compose.yml` and protected mode-600 `.env` are covered by the generic Compose-file/`.env` collection. The account password was changed in-app; the username/password values retained in `.env` are superseded historical first-run records only and are not injected into the container.
  - `/home/patryk/docker/watcharr/CREDENTIALS.md` is explicitly archived with the Compose definition and contains no credentials.
  - `/home/patryk/docker/watcharr/data` is explicitly copied under `bind_mounts/watcharr_data/`; this includes `watcharr.db`, `watcharr.json`, cache, and any adjacent persistent application state.
  - The backup script briefly stops Watcharr before copying the full data tree, as upstream requires for SQLite consistency, and its EXIT safety trap restarts the service if a later backup step fails.
  - Restore verification requires the Compose file, protected environment, SQLite database, and server configuration.
- ConvertX:
  - `/home/patryk/docker/convertx/compose.yml` and protected mode-600 `.env` are covered by the generic Compose-file/`.env` collection. The account email/password values retained in `.env` are superseded historical first-run records only; current login state is stored in SQLite and managed in-app.
  - `/home/patryk/docker/convertx/CREDENTIALS.md` is explicitly archived with the Compose definition and contains no credentials.
  - `/home/patryk/docker/convertx/data` is archived through a short-lived root container under `bind_mounts/convertx_data/data.tar.gz`; this preserves the SQLite database, uploads, converted outputs, and job history without broadening the live root-owned mode-750 directory.
  - Restore verification requires the Compose file, protected environment, and persistent-data archive.
- Reverse proxy:
  - `/home/patryk/docker/reverse-proxy/compose.yml`
  - `/home/patryk/docker/reverse-proxy/Caddyfile`
  - archived under `compose/reverse-proxy/`
- Caddy (DNS-01 TLS):
  - `/home/patryk/docker/caddy/compose.yml`
  - `/home/patryk/docker/caddy/Caddyfile`
  - `/home/patryk/docker/caddy/Dockerfile`
  - `/home/patryk/docker/caddy/.env` (stored without printing secret values; holds `CLOUDFLARE_API_TOKEN`)
  - archived under `compose/caddy/`
- Shlink:
  - `/home/patryk/docker/shlink/compose.yml`, `/home/patryk/docker/shlink/Caddyfile.public`, and `/home/patryk/docker/shlink/Dockerfile.public`
  - `/home/patryk/docker/shlink/.env` (mode 600; stored without printing the database password or admin API key)
  - PostgreSQL is dumped online from `shlink-db` in custom archive format to `bind_mounts/shlink/shlink-db.dump`
  - the dump must pass `pg_restore --list` before archive creation; the live PostgreSQL data directory is deliberately not copied while the database is running
  - the restore verifier requires the Compose, protected environment, gateway configuration/build file, and logical database dump
- n8n:
  - `/home/patryk/docker/n8n/docker-compose.yml`
  - `/home/patryk/docker/n8n/.env`
  - `/home/patryk/docker/n8n/data`
  - archived under `compose/n8n/` and `bind_mounts/n8n_data/`
- UpSnap:
  - `/home/patryk/docker/upsnap/compose.yml`
  - `/home/patryk/docker/upsnap/data` (PocketBase `pb_data` bind mount)
  - archived under `compose/upsnap/` and `bind_mounts/upsnap_data/`
- Paperless-ngx:
  - `/home/patryk/docker/paperless/docker-compose.yml`
  - `/home/patryk/docker/paperless/.env`
  - PostgreSQL database (dumped via `pg_dump` from `paperless-db` to `db_dump.sql` before archiving)
  - `/home/patryk/Documents/paperless/media`
  - `paperless_data` Docker volume
  - archived under `compose/paperless/`, `bind_mounts/paperless/` (including `db_dump.sql` and `media/`), and `volumes/paperless/`
  - `consume` and `export` folders are not backed up
- Backup script:
  - `/home/patryk/scripts/homelab-backup.sh`
- Restore verification script:
  - `/home/patryk/scripts/homelab-restore-verify.sh`
- Canonical documentation mirror synchronization:
  - `/home/patryk/scripts/sync-homelab-md-mirror.sh`
  - `/home/patryk/.config/systemd/user/homelab-md-mirror-sync.service`
  - `/home/patryk/.config/systemd/user/homelab-md-mirror-sync.path`
  - `/home/patryk/.config/systemd/user/homelab-md-mirror-sync.timer`
- NAS mount alert script and push URL file:
  - `/home/patryk/scripts/nas-mount-check.sh`
  - `/home/patryk/scripts/uptime-kuma-nas-mount-push.url`
- Homelab repo (including its `.git` directory, which holds the full commit history):
  - `/home/patryk/homelab-repo` (archived in full, `.git` included, under `bind_mounts/homelab_repo/`)
  - `/home/patryk/scripts/collect-homelab-repo.sh`
  - `/home/patryk/.config/systemd/user/homelab-repo-collect.service`
  - `/home/patryk/.config/systemd/user/homelab-repo-collect.timer`
- Forgejo (the Git server application itself, separate from the `homelab-repo` snapshot it hosts):
  - `/home/patryk/docker/forgejo/compose.yml` (archived under `bind_mounts/forgejo/`)
  - `/home/patryk/docker/forgejo/data` — SQLite database, repos, `app.ini`, and root-owned host SSH keys — archived via `docker run` (root inside the container can read the root-owned `ssh/` subdirectory) as `bind_mounts/forgejo_data/data.tar.gz`
  - `/home/patryk/.ssh/forgejo-homelab-repo-deploy` and its `.pub`, the deploy key used for `homelab-repo`'s push, archived under `ssh_keys/` with `cp -a` (mode 600 preserved, contents never printed)
- Public GitHub docs push:
  - `/home/patryk/scripts/push-homelab-docs-public-github.sh`
  - `/home/patryk/.config/systemd/user/homelab-docs-public-github-push.service`
  - `/home/patryk/.config/systemd/user/homelab-docs-public-github-push.path`
  - `/home/patryk/.config/systemd/user/homelab-docs-public-github-push.timer`
  - `/home/patryk/.ssh/github-homelab-docs-public-deploy` and its `.pub`, archived under `ssh_keys/` with `cp -a` (mode 600 preserved, contents never printed), mirroring the Forgejo deploy key above
  - The `/home/patryk/homelab-docs-public-github` working copy is intentionally not archived: it holds no unique state, only a copy of `HOMELAB.md` (already backed up under `docs/`) plus commit history that also exists on GitHub, and it is reconstructed with `git clone` plus one push.
- miniPC services diagram push:
  - `/home/patryk/scripts/generate-push-minipc-diagram.sh`
  - `/home/patryk/.config/systemd/user/minipc-diagram-push.service`
  - `/home/patryk/.config/systemd/user/minipc-diagram-push.path`
  - `/home/patryk/.config/systemd/user/minipc-diagram-push.timer`
  - No separate deploy key: this script reuses the same `github-homelab-docs-public` deploy key already archived under the Public GitHub docs push entry above.
  - The `/home/patryk/homelab-docs-public-github` working copy still needs no dedicated backup coverage for the same reason stated above: `diagrams/minipc.md` is generated fresh from `HOMELAB.md` (itself already backed up under `docs/`) on every run, and the working copy's commit history also exists on GitHub, so it remains fully reconstructible via `git clone` plus one push.
- Mac mini iCloud documentation delivery:
  - `/home/patryk/scripts/push-homelab-macmini-icloud.sh`
  - `/home/patryk/.config/systemd/user/homelab-macmini-icloud-push.service`
  - `/home/patryk/.config/systemd/user/homelab-macmini-icloud-push.path`
  - `/home/patryk/.config/systemd/user/homelab-macmini-icloud-push.timer`
  - `/home/patryk/.ssh/macmini-icloud-push-deploy` and its `.pub`, archived under `ssh_keys/` with `cp -a` (mode 600 preserved, contents never printed)

- Daily Homelab Report:
  - `/home/patryk/scripts/daily-homelab-report.sh`
  - `/etc/systemd/system/daily-homelab-report.service` if installed
  - `/etc/systemd/system/daily-homelab-report.timer` if installed
  - `/home/patryk/scripts/ram-sample.sh`
  - `/home/patryk/.config/systemd/user/ram-sample.service`
  - `/home/patryk/.config/systemd/user/ram-sample.timer`
- TLS certificate expiry checker:
  - `/home/patryk/scripts/tls-cert-expiry-check.sh`
  - `/home/patryk/.config/systemd/user/tls-cert-expiry-check.service`
  - `/home/patryk/.config/systemd/user/tls-cert-expiry-check.timer`
- Reboot Needed notifier:
  - `/home/patryk/scripts/reboot-needed-check.sh`
  - `/home/patryk/scripts/reboot-needed-check.state` if it exists
  - `/etc/systemd/system/reboot-needed-check.service` if installed
  - `/etc/systemd/system/reboot-needed-check.timer` if installed
- Boot notification:
  - `/home/patryk/scripts/boot-notify.sh`
  - `/etc/systemd/system/boot-notify.service` if installed
- Anti-sleep monitor:
  - `/home/patryk/scripts/anti-sleep-check.sh`
  - `/home/patryk/scripts/anti-sleep-check.baseline`
  - `/home/patryk/scripts/anti-sleep-check.last.log`
  - `/etc/systemd/system/anti-sleep-check.service`
  - `/etc/systemd/system/anti-sleep-check.timer`
- SMART health checker:
  - `/home/patryk/scripts/smart-health-check.sh`
  - `/etc/systemd/system/smart-health-check.service`
  - `/etc/systemd/system/smart-health-check.timer`
  - The live `/home/patryk/scripts/smart-health-check-state/` anti-spam baseline is intentionally excluded: it is transient operational state, not required for reconstruction, and restoring stale alert suppression could hide a new event.
- Trivy image vulnerability scanner:
  - `/home/patryk/scripts/trivy-image-scan.sh`
  - `/etc/systemd/system/trivy-image-scan.service`
  - `/etc/systemd/system/trivy-image-scan.timer`
  - The live `/home/patryk/scripts/trivy-image-scan-state/` baseline and lock are intentionally excluded: they are transient alert-suppression state, and recovery resets them fresh so stale suppression cannot hide a new finding.
- Paperless automation:
  - `/home/patryk/scripts/paperless-consistency-check.sh`
  - `/home/patryk/scripts/paperless-consistency-check-notify.sh`
  - `/home/patryk/scripts/paperless-autofill-document-type.sh`
  - `/home/patryk/scripts/paperless-sync-title-to-filename.sh`
  - `/home/patryk/.config/systemd/user/paperless-consistency-check.service`
  - `/home/patryk/.config/systemd/user/paperless-consistency-check.timer`
  - `/home/patryk/.config/systemd/user/paperless-title-sync.service`
  - `/home/patryk/.config/systemd/user/paperless-title-sync.timer`
- systemd units:
  - `/etc/systemd/system/homelab-backup.service`
  - `/etc/systemd/system/homelab-backup.timer`
  - `/etc/systemd/system/kuma-tailscale-push.service`
  - `/etc/systemd/system/kuma-tailscale-push.timer`
  - `/etc/systemd/system/nas-mount-check.service`
  - `/etc/systemd/system/nas-mount-check.timer`
  - `/etc/systemd/system/daily-homelab-report.service`
  - `/etc/systemd/system/daily-homelab-report.timer`
  - `/etc/systemd/system/anti-sleep-check.service`
  - `/etc/systemd/system/anti-sleep-check.timer`
- anti-sleep systemd config drop-ins:
  - `/etc/systemd/logind.conf.d/99-disable-suspend.conf`
  - `/etc/systemd/sleep.conf.d/99-disable-all-sleep.conf`
  - archived under `systemd_config/logind.conf.d/`
  - archived under `systemd_config/sleep.conf.d/`
- Samba config:
  - `/etc/samba/smb.conf`
  - archived under `system_config/samba/`
- NAS copy to `/mnt/nas-minipc-backup`
- Local mixed retention: keeps the newest 5 backup archives, plus one archive closest to 7, 14, 21, 30, and 60 days ago
- NAS mixed retention: keeps the newest 5 backup archives, plus one archive closest to 7, 14, 21, 30, and 60 days ago
- Uptime Kuma push notification after successful backup

Intentionally not backed up:


- temporary folders created during backup
- old backup archives beyond retention
- backup archives outside the mixed retention policy
- Fedora Web Console / Cockpit: `https://192.168.10.12:9090/`
- Fedora Web Console / Cockpit hostname/LAN alternative: `https://fedora:9090/`
- secret file contents printed to terminal
- Uptime Kuma push URL contents

## Restore verification

The restore verification script is:

```bash
/home/patryk/scripts/homelab-restore-verify.sh
```

Purpose:

- Safely validates the newest backup by running a tar listing, extracting the archive into `/tmp`, and checking that important recovery files are present.
- Never restores over live paths and does not write into `/home/patryk/docker`, `/home/patryk/scripts`, `/etc`, Docker volumes, or any other live restore location.
- Uses invocation-unique `mktemp` paths. Successful verification removes its extraction directory and tar listing automatically; failed verification preserves them for diagnosis. Use `--keep-temp` to retain them after a successful run.

Usage:

```bash
/home/patryk/scripts/homelab-restore-verify.sh
/home/patryk/scripts/homelab-restore-verify.sh /path/to/archive.tar.gz
/home/patryk/scripts/homelab-restore-verify.sh --keep-temp [archive.tar.gz]
```

Output includes:

- PASS/FAIL summary
- archive verified
- restore test directory
- tar listing file
- required recovery item counts and missing items

Recommended use:

- Run manually after major service changes.
- Run manually after changing backup coverage.

## Samba / SMB file access

Samba is enabled for authenticated file access from iPhone Files, Samsung Fold, and Mac.

Current working SMB URLs:

```text
smb://192.168.10.12
smb://192.168.10.12/backups
smb://192.168.10.12/documents
smb://192.168.10.12/downloads
```

The server-root URL `smb://192.168.10.12` shows the share list. The direct share URLs map to existing folders under `/home/patryk`; `downloads` maps to `/home/patryk/downloads`.

Security and access:

- Authenticated user only: `patryk`
- Guest access: disabled
- SMB binds only to `lo` and LAN interface `enp1s0`
- Preferred access is through the LAN IP `192.168.10.12`
- SMB password is managed with `smbpasswd` and is not stored in this document
- Apple SMB compatibility options (`vfs objects = catia fruit streams_xattr` and related `fruit` metadata/resource fork settings) are enabled on all Samba shares whose paths are under `/home/patryk`. This allows the native iOS/macOS Apple Files app to see shares and folders as fully writable by properly handling AppleDouble metadata. This is applied on a per-share basis, not globally, and is not a permission broadening like `chmod 777`.
- Do not expose SMB publicly
- The `SMB1 disabled -- no workgroup available` message from `smbclient` is harmless
- Server-root browsing required explicit protocol settings in `/etc/samba/smb.conf`:
  - `server min protocol = SMB2_10`
  - `server max protocol = SMB3`
- `testparm` may display the minimum protocol as `SMB2`; this is acceptable normalization.
- Android SMB clients can browse `smb://192.168.10.12` and show all shares correctly.
- Apple iOS/macOS Files/Finder compatibility remains imperfect even though Android and `smbclient` work. The following Apple compatibility settings do not fully resolve it:
  - `vfs objects = streams_xattr`
  - `vfs objects = fruit streams_xattr`
- For iOS/macOS, prefer direct share URLs from clients that support them correctly, or consider a third-party SMB-capable file manager later instead of changing Samba further.

Notes:

- SELinux is currently disabled at runtime on the host; `/etc/selinux/config` remains set to `SELINUX=enforcing`, but no SELinux policy is loaded in the running system. `samba_enable_home_dirs` is retained for home-directory sharing support if SELinux is later active.
- `samba_export_all_rw` is intentionally not enabled.
- Directly sharing the root of `/home/patryk` as `patryk-home` does not work and returns `NT_STATUS_ACCESS_DENIED listing *`, even though authentication succeeds and existing subfolder shares work.
- The current configuration uses separate shares for existing folders over localhost and LAN IP.
- Future rule: new folders are not automatically added to SMB. To expose another folder later, add a new share to `/etc/samba/smb.conf`; with SELinux Enforcing, non-home paths may also need an SELinux file context such as `samba_share_t` via `semanage fcontext` and `restorecon`.

## Container update policy / DIUN

DIUN (Docker Image Update Notifier) is configured to monitor running containers and report findings via the Daily Homelab Report; it does not send separate notifications.

**Important DIUN rules:**
- **Role:** Detection only. It detects new Docker image versions and findings are included in the daily report.
- **Daily report relationship:** The Daily homelab report reads DIUN image data for Docker image update reporting. Fedora/system update reporting in the Daily homelab report is separate and comes from `dnf`/`dnf5` checks, not from DIUN.
- **Auto-update:** None. DIUN does not auto-update containers.
- **Notifications:** Avoid duplicate standalone Docker update notifications unless explicitly requested.
- **Cadence:** Manual, planned maintenance windows. No spontaneous updates.

**Pre-update checklist:**
1. Read the daily report container update summary.
2. Identify the affected service.
3. Check release notes or changelog (especially for major version bumps).
4. Run homelab backup: `/home/patryk/scripts/homelab-backup.sh`
5. Verify backup completed successfully.

**Update command pattern (documentation only, do not run blindly):**
```bash
cd /home/patryk/docker/service-name
docker compose pull
docker compose up -d
```

**Post-update checklist:**
1. Check container status: `docker ps`
2. Check container logs: `docker logs --tail=80 container-name`
3. Test the service URL and UI.
4. Update `HOMELAB.md` if the version, config, behavior, backup coverage, or service URLs/ports changed.
5. Run restore verify: `/home/patryk/scripts/homelab-restore-verify.sh`

**Rollback procedure:**
- Stop if the update fails.
- Check logs.
- Revert the `docker-compose.yml` to the previous specific image tag (e.g., `image: my-service:v1.2.3` instead of `:latest`).
- Run `docker compose up -d` to rollback.
- Do not blindly run volume or image pruning.

**Safety rules:**
- NO `docker system prune` or `docker volume prune` as part of updates.
- NO mass updates of the whole stack at once. Update only the selected service.
- NO Fedora/dnf system updates during container maintenance.
- High-risk services (FreshRSS, Paperless, reverse-proxy, Samba configs, backup scripts) require careful checking.
- Low-risk services (Homarr, changedetection) can be updated with simpler checks if there is no major data impact.
- Always backup before and restore verify after.

## Known issues / watch items

- changedetection.io has no Playwright/browser helper yet, so JavaScript-heavy pages may need that later.
- Apple iOS/macOS SMB compatibility is still imperfect; Android SMB browsing works.
- GNOME Remote Desktop/RDP is configured, but previous headless/HDMI issues should be treated as a known risk if they return.
- Router DHCP DNS now uses primary `192.168.10.12` and secondary `192.168.10.13`; because clients may use both, the two AdGuard Home instances should stay mirrored. Emergency DNS recovery is still manual through OpenVPN to the Cudy router.

## Useful commands

Show this file:

```bash
less ~/docker/HOMELAB.md
```

Run backup manually:

```bash
/home/patryk/scripts/homelab-backup.sh
```

Run restore verification manually:

```bash
/home/patryk/scripts/homelab-restore-verify.sh
```

Check backup timer:

```bash
systemctl status homelab-backup.timer --no-pager -l
systemctl status homelab-backup.service --no-pager -l
```

Check latest local backups:

```bash
ls -lh /home/patryk/backups/homelab | tail
```

Check latest NAS backups:

```bash
ls -lh /mnt/nas-minipc-backup | tail
```

Check Docker containers:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

Check Docker stats:

```bash
docker stats --no-stream
```

Check listening ports:

```bash
ss -tulpn | grep -E ':(22|53|80|443|3001|3002|5000|5678|7575|8010|8082|8083|8095|8096|8181|3389|9090)\b' || true
```

Check NAS mount:

```bash
mountpoint /mnt/nas-minipc-backup
df -h /mnt/nas-minipc-backup
```

Check Samba:

```bash
testparm -s /etc/samba/smb.conf
systemctl status smb --no-pager
ss -lntup | grep -E ':(139|445)' || true
getsebool samba_enable_home_dirs
```

Check RAM:

```bash
free -h
cat /proc/meminfo | grep -E 'MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|SReclaimable|Shmem'
```

Check logs for a Docker container:

```bash
docker logs --tail=80 CONTAINER_NAME
```

Check Mac mini Glances CPU/RAM from the Fedora miniPC:

```bash
curl -fsS http://192.168.10.13:61208/api/4/cpu
curl -fsS http://192.168.10.13:61208/api/4/mem
```

## Paperless Workflow

Paperless-ngx is configured for a simplified document archiving workflow for CMR documents, UK customs/export documents, and fuel/transport/parking invoices.

- **Import path (SMB):** `//192.168.10.12/documents/paperless/consume`
- **Document Types:** `CMR`, `Dokument celny`, `Faktura`
- **Document Type matching:** Document Types do not classify by OCR directly. Storage Path matchers identify the archive path, and the consistency automation fills a missing Document Type from the assigned Storage Path.
- **Filename prefix classification (implemented 2026-07-30):** Three enabled Paperless Workflows now classify newly consumed documents on the `Document Added` event, using the saved `original_filename` rather than the current title. This behavior had previously been documented here as if it existed, but there was no Paperless Workflow or consume script implementing it until 2026-07-30. Treat documentation as intended state and verify deployed workflow state when troubleshooting.
  - Paperless-ngx `5.2.16` implements `filter_filename` as case-insensitive Python `fnmatch` against the complete original basename, including its extension. The separate Workflow `matching_algorithm = Regular expression` option does not apply to filenames.
  - Workflow `Filename prefix - CMR` (Workflow ID `2`, Trigger IDs `1` and `2`) uses the OR-ed filename patterns `CMR.pdf` and `CMR[ _-]*`, then assigns Document Type `CMR` and Storage Path `CMR`.
  - Workflow `Filename prefix - UK customs` (Workflow ID `3`, Trigger IDs `3` and `4`) uses the OR-ed filename patterns `UK.pdf` and `UK[ _-]*`, then assigns Document Type `Dokument celny` and Storage Path `CMR`.
  - Workflow `Filename prefix - FAKTURA` (Workflow ID `4`, Trigger IDs `5` and `6`) uses the OR-ed filename patterns `FAKTURA.pdf` and `FAKTURA[ _-]*`, then assigns Document Type `Faktura` and Storage Path `FAKTURY`.
  - Required matches include `CMR.pdf`, `CMR 1.pdf`, `CMR_Amazon.pdf`, `UK.pdf`, `UK customs.pdf`, `UK_01.pdf`, `FAKTURA.pdf`, `FAKTURA parking.pdf`, and `FAKTURA_Deutschland.pdf`. Required non-matches include `CMRsomething.pdf`, `UKRAINA.pdf`, and `FAKTURAX.pdf`.
  - Glob limitation: the delimiter patterns safely cover space, underscore, or hyphen followed by any remainder/extension. End-of-stem is represented safely only for bare `.pdf` names (`CMR.pdf`, `UK.pdf`, and `FAKTURA.pdf`). A bare non-PDF name such as `CMR.jpg` does not match, even though the ideal stem rule would accept it. This deliberate under-match avoids unsafe dot-based globs that would also accept stems such as `CMR.something`.
  - Safety: every trigger excludes all currently configured Document Types (`CMR`, `Dokument celny`, `Faktura`) and Storage Paths (`CMR`, `FAKTURY`). Therefore the assignment runs only when both fields are unset, and a manually or otherwise correctly classified new document is not overwritten. Paperless `5.2.16` has no direct "assign only if unset" action option; if a new Document Type or Storage Path is added, add it to all six trigger exclusion lists to preserve this guarantee. The consistency checker reports prefix/metadata conflicts separately.
  - The prefix is only a classification instruction. The normal title/archive synchronization later replaces the incoming filename with the documented final `DD-MM-YYYY - 000000x` naming convention.
- **Tags:** `paliwo`, `transport`, `rozliczenie`
- **Correspondents:** Correspondents are intentionally not used.
- **Document Date:** Document date should be set as the Paperless created date.
- **Archive Filename & UI Title:** Paperless archive filenames are based on the Created date plus a safe unique suffix (the 7-digit Document ID `{{ doc_pk }}`). The Paperless UI title is automatically synchronized to match the archive filename (without `.pdf`), e.g., `03-07-2026 - 0000004`. UK customs documents additionally use ` - UK`.
  - **Sync Script:** The UI Title is automatically synchronized twice daily (06:30 and 18:30) via a systemd user timer (`paperless-title-sync.timer`) which triggers `/home/patryk/scripts/paperless-sync-title-to-filename.sh`. This ensures the UI title securely matches the physical file without manually renaming.
  - **Manual Trigger:** If you need to force a sync immediately, run `systemctl --user start paperless-title-sync.service`.
  - **Check Schedule:** Check the timer status using `systemctl --user list-timers paperless-title-sync.timer`.
- **Storage Paths Strategy:** Paperless uses storage paths with numeric year-month folders for reliability instead of Polish month names:
  1. **CMR**
     - Path: `CMR/{{ created_year }}-{{ created_month }}/{{ title }}`. The title synchronizer supplies `DD-MM-YYYY - 000000x` for normal CMR and appends ` - UK` for Document Type `Dokument celny`.
     - Document Type: `CMR` or `Dokument celny`
     - Matching pattern: `handover goods Carrier reservation Frachtfahrer "Number of packages"` (all terms/phrase, case-insensitive)
     - Example: `CMR/2026-07/03-07-2026 - 0000123.pdf`
  2. **UK customs/export documents**
     - Storage Path: `CMR` (the same physical `CMR/YYYY-MM/` folder as normal CMR documents)
     - Document Type: `Dokument celny`
     - Matching pattern: `MRN Exportateur Destinataire "BUREAU DE DEPART" "Pays de destination"` (all terms/phrases, case-insensitive) in the bounded autofill OCR fallback.
     - The Document Type adds the ` - UK` title and archive filename suffix: `CMR/2026-07/10-07-2026 - 0000124 - UK.pdf`.
  3. **FAKTURY**
     - Path: `FAKTURY/{{ created_year }}-{{ created_month }}/{{ created_day }}-{{ created_month }}-{{ created_year }} - {{ doc_pk }}`
     - Document Type: `Faktura`
     - Matching pattern: regex, case-insensitive: `(?s)\b(Faktura|paliwo|invoice|Rechnung)\b|\b(mastercard\s+debit|cardholder\s+copy)\b.{0,1000}\bovernight\s+parking\b.{0,500}\bVAT\b|\bovernight\s+parking\b.{0,500}\bVAT\b.{0,1000}\b(mastercard\s+debit|cardholder\s+copy)\b`
     - Parking receipts require multiple signals: parking plus card/payment wording plus VAT. Do not classify parking documents from a merchant name, address, card number, transaction ID, or a generic amount/date alone.
     - Example: `FAKTURY/2026-07/03-07-2026 - 0000125.pdf`
- **Autofill fallback:** On `Document Added`, a matching original-filename workflow assigns the prefix mapping only if Document Type and Storage Path are still unset. If either field is already set, the workflow guard skips assignment and the consistency checker remains responsible for reporting any prefix/metadata conflict. Files left unresolved continue through the bounded customs OCR condition or existing safe Storage Path/OCR matching, then Storage Path-to-Document Type autofill; anything still unresolved remains visible to the consistency checker. Customs documents must not be assigned Document Type `CMR`.
- **Consistency rules:** `CMR` type and `Dokument celny` type both use Storage Path `CMR`; `Faktura` type uses Storage Path `FAKTURY`. The checker also validates original-filename prefix intent when present, flags customs-like content typed as `CMR`, CMR-like content typed as customs, and invoice-like content assigned outside `Faktura`/`FAKTURY`. The ` - UK` title/archive suffix is required only for Document Type `Dokument celny` and is rejected for normal `CMR`.
- **Accepted consistency-check exceptions:** Document IDs `15`, `19`, and `20` are explicitly excluded from the consistency checker. Their original source files are permanently missing and no recoverable backup exists; the owner accepted this on `2026-07-31` because each archive PDF remains valid and usable. This is an ID-only allowlist, not a general missing-original-file suppression; all other documents remain subject to every consistency check.
- **Manual assignment/correction:** If Paperless fails to auto-detect the type, storage path, or date correctly upon import, correct the Document Type, Storage Path, and Created date manually in the UI. Unrecognized documents should remain visible as consistency-check issues until they are manually classified or intentionally handled.
- **Note:** Paperless may convert uploaded photos/images into archived PDFs.
- **SMB Usage & Permissions:** Paperless `consume` is the supported upload/drop folder. Paperless `media` directories (`archive`, `originals`, `thumbnails`) are normalized to `patryk:patryk` with setgid `2775` directories and `0664` files for SMB browse compatibility. They may be browsed over SMB, but should not be manually edited, moved, or deleted.

## Maintenance rule

HOMELAB.md is the source of truth for the miniPC homelab. Old chats are context only.

This file intentionally contains only current-state information. Update it only when the current state changes, such as a new service, changed URL or port, changed path, changed backup coverage, or changed operating policy. Do not append dated narratives merely to record that maintenance or verification occurred.

Detailed change history belongs in Git commit messages. The `homelab-repo` collector automatically snapshots the approved homelab files and pushes the repository to Forgejo.

When the current state changes, keep this file updated with:

- service name
- URL/port
- Docker folder or systemd unit
- backup status
- Uptime Kuma monitoring status if relevant
- notes about data/volumes/config

## Canonical documentation mirror synchronization

- Purpose: maintain an SMB-accessible copy of the canonical homelab documentation so it can be read or transferred through the existing SMB share.
- Canonical source: `/home/patryk/docker/HOMELAB.md`.
- SMB-readable mirror destination: `/home/patryk/Documents/homelab/HOMELAB.md`.
- Synchronization is strictly one-way from the canonical source to the SMB-readable mirror. The mirror must never be treated as the source of truth, and reverse synchronization from the SMB copy is not allowed.
- Script: `/home/patryk/scripts/sync-homelab-md-mirror.sh`. It writes the destination in place and copies only when the files differ.
- Event-driven synchronization is provided by `homelab-md-mirror-sync.path`; fallback synchronization is provided by `homelab-md-mirror-sync.timer` every 15 minutes.
- Systemd user units:
  - `/home/patryk/.config/systemd/user/homelab-md-mirror-sync.service`
  - `/home/patryk/.config/systemd/user/homelab-md-mirror-sync.path`
  - `/home/patryk/.config/systemd/user/homelab-md-mirror-sync.timer`
- The mechanism contains no credentials or secrets.

Useful checks:

```bash
systemctl --user status homelab-md-mirror-sync.path --no-pager
systemctl --user status homelab-md-mirror-sync.timer --no-pager
systemctl --user status homelab-md-mirror-sync.service --no-pager
systemctl --user list-timers --all --no-pager | grep -F homelab-md-mirror-sync
cmp -s /home/patryk/docker/HOMELAB.md /home/patryk/Documents/homelab/HOMELAB.md
```

Manual one-shot synchronization only, not normal routine operation:

```bash
systemctl --user start homelab-md-mirror-sync.service
```

## Desktop session cleanup

- GNOME Software autostart is disabled only for user `patryk` through user override file `/home/patryk/.config/autostart/org.gnome.Software.desktop` to reduce background RAM usage in the GNOME session.
- GNOME Software remains installed and can still be launched manually when needed with `gnome-software &`.
- Re-enable autostart by deleting `/home/patryk/.config/autostart/org.gnome.Software.desktop`.

After major service, backup, storage, network, or alerting changes, run the homelab backup and restore verification when appropriate.

## Changelog

Detailed change history is tracked in this host's Git repository (see the Homelab repo entries above), pushed to Forgejo. This file intentionally contains only current-state information.

## Planned services / ideas

### Current direction

- Homarr is the active miniPC landing page. The approved board is the single source for dashboard layout changes.
- CPU/RAM monitoring is completed through the active Glances integration; do not alter the approved widget configuration as part of routine dashboard maintenance.

### Planned later

- AI and the Discord bot are hosted on the Mac mini and are no longer planned Fedora miniPC services.
- MeTube is hosted on the Mac mini, not Fedora. Do not restore it on the miniPC. Homarr links to the Mac mini instance at `http://192.168.10.13:8091`.
- Taildrop / Tailscale file access refinements: only if needed later.
- Test restore from backup: still worth doing later as a dedicated maintenance task.

### Rejected / not planned

- CasaOS: not planned. The current Fedora + Docker Compose + Homarr + `HOMELAB.md` setup is intentional, and CasaOS would add an extra management layer that could conflict with the controlled setup.
- Dozzle / Dockge: not planned right now. Terminal and AI-assisted diagnostics are sufficient for now, so a separate web log / compose management layer is optional and not worth adding yet.
- Duplicate n8n Paperless watchdog: not planned right now because Paperless consistency checking already exists as a systemd timer.
- Optional Paperless consume stuck-file checker: not planned right now.

## Later Maintenance

1. Fedora/security/system updates
2. Docker/container updates such as Speedtest Tracker if still reported by DIUN / Daily Homelab Report

### Fedora update caution

- Before Fedora/system updates: verify backup status, restore verification readiness, SSH access, Fedora Web Console / Cockpit access, AdGuard/DNS health, and sleep-target masks.
- During planned Fedora/system maintenance, temporary router alternate DNS such as `1.1.1.1` is allowed if needed to preserve client internet access while the miniPC or AdGuard Home is down.
- After Fedora/system updates and after any reboot: verify AdGuard/DNS, Docker service health, SSH access, and sleep/suspend/hibernate masks again.
- Do not assume anti-sleep protection survived a kernel or system update until the masks and related settings are checked again.

Already implemented and active: changedetection.io, Wallos, SnapOtter, Paperless-ngx, Pushover backup/homelab alerts, Daily homelab report, Reboot-needed notifier, Speedtest Tracker, DIUN, Anti-sleep monitoring, and NAS mount alert.
