# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PassWall2 is a LuCI web interface application (`luci-app-passwall2`) for OpenWrt routers that provides multi-protocol proxy and VPN management. It is packaged as an OpenWrt feed package and deployed to routers.

All application code lives under `luci-app-passwall2/`.

## Building

This package is built using the OpenWrt build system (not standalone). To build:

```bash
# From within an OpenWrt SDK or buildroot with this feed configured:
make package/luci-app-passwall2/compile
```

The `Makefile` uses `luci.mk` from the OpenWrt LuCI feed. Current version is defined in `luci-app-passwall2/Makefile` (`PKG_VERSION`).

## Architecture

The app follows the standard OpenWrt LuCI package structure:

### `luasrc/` — Lua source (runs on the router's LuCI web server)

- **`controller/passwall2.lua`** — Route definitions. Registers all LuCI menu entries and API endpoints under `/admin/services/passwall2/`.
- **`passwall2/api.lua`** — Shared utility library. Defines `appname`, path constants (`TMP_PATH`, `CACHE_PATH`, `LOG_FILE`), and helper functions used throughout Lua code.
- **`passwall2/com.lua`** — Common protocol/node detection helpers.
- **`passwall2/util_xray.lua`**, **`util_sing-box.lua`**, **`util_shadowsocks.lua`**, **`util_hysteria2.lua`**, **`util_naiveproxy.lua`**, **`util_tuic.lua`** — Per-protocol config generators. Each generates the JSON/YAML config file for that proxy binary.
- **`passwall2/server_app.lua`** — Server-side proxy application logic.
- **`model/cbi/passwall2/client/`** — LuCI CBI form definitions for each client settings page (global, node list, subscriptions, ACL, shunt rules, etc.).
- **`model/cbi/passwall2/client/type/`** — Protocol-specific node configuration forms (xray/ray, sing-box, ss, ssr, hysteria2, naive, tuic, ss-rust).
- **`model/cbi/passwall2/server/`** — Server mode forms.
- **`view/passwall2/`** — HTM template fragments used by CBI forms.

### `root/` — Files deployed directly to the router filesystem

- **`usr/share/passwall2/app.sh`** — Main service orchestration script. Starts/stops proxy processes, sets up iptables/nftables transparent proxy rules.
- **`usr/share/passwall2/utils.sh`** — Shell utility functions sourced by other scripts.
- **`usr/share/passwall2/subscribe.lua`** — Lua script that parses subscription URLs and imports nodes into UCI config.
- **`usr/share/passwall2/helper_dnsmasq.lua`** — Configures dnsmasq for DNS-based routing.
- **`usr/share/passwall2/iptables.sh`** / **`nftables.sh`** — Firewall rule management for transparent proxy (tproxy).
- **`usr/share/passwall2/monitor.sh`** — Process monitor for auto-restart.
- **`usr/share/passwall2/haproxy.lua`** — Generates HAProxy config for load balancing.
- **`usr/share/passwall2/rule_update.lua`** — Updates routing rule files (geoip/geosite).
- **`etc/init.d/passwall2`** / **`passwall2_server`** — OpenWrt init scripts (start/stop/restart).
- **`etc/config/passwall2_server`** — Default UCI config for server mode.
- **`etc/hotplug.d/iface/98-passwall2`** — Restarts service on network interface changes.

### `htdocs/` — Static assets

- **`luci-static/resources/view/passwall2/`** — JavaScript libraries (Sortable.min.js for node list drag-and-drop, qrcode.min.js for QR code sharing).

### `po/` — Translations

Language files: `zh-cn/`, `zh-tw/`, `fa/` (Persian), `ru/`.

## Key Concepts

**UCI Config**: All settings are stored in `/etc/config/passwall2` using OpenWrt's UCI system. The `api.uci` cursor is used throughout Lua code.

**Transparent Proxy**: The app supports both iptables (fw3/legacy) and nftables (fw4) backends. `app.sh` auto-detects which is available at runtime.

**Protocol Util Files**: When adding or modifying support for a proxy protocol, the corresponding `util_*.lua` in `luasrc/passwall2/` generates the runtime config, and `model/cbi/passwall2/client/type/*.lua` provides the UI form.

**Subscription Parsing**: `subscribe.lua` runs as a standalone Lua script (not inside LuCI) and writes nodes directly to UCI config.

## Logs (on router)

- Main log: `/tmp/log/passwall2.log`
- Server log: `/tmp/log/passwall2_server.log`
- Temp configs: `/tmp/etc/passwall2/`
