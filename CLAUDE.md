# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

OpenWrt all-in-one package: **dae** (kernel/core) + **daed** (companion) + **luci-app-daede** (LuCI management UI). Active work is the **luci-app-daede LuCI frontend**.

- This repo is the **upstream** for `dae`, `daed`, and `luci-app-daede`. They build from an aggressive pinned source (daeuniverse + olicesx perf forks mirrored to kenzok8), assembled into hosted `dae-src`/`daed-src` release tarballs and verified by `PKG_HASH`. Pins live in `ci/pins.env`; the assemble + auto-bump workflows manage them. Downstream feeds (wall/jell/small/small-package) consume dae/daed/luci-app-daede from here — `dae/`, `daed/` carry only `Makefile` + `files/`.
- This repo's own development surface is `luci-app-daede/`.

## luci-app-daede architecture (the part you'll edit)

Client-side LuCI views only — there is **no custom rpcd/ubus backend**. The UI drives the system through LuCI's stock `fs` / `fs.exec` / `uci` RPCs, and a `proxy-check` etc. via helper scripts.

- **Views** (`luci-app-daede/htdocs/luci-static/resources/view/daede/`): `config.js` (the large one, ~1100 lines), `updates.js`, `log.js`, and `backend.js`. `backend.js` is a shared `baseclass` module (`'require view.daede.backend as backend'`) holding backend detection + service status; the other three import it.
- **Two interchangeable backends**: `dae` (config-file based) and `daed` (web dashboard), selected via uci `daede.config.active_backend`. `backend.js` detects which are installed/running.
- **Privileged work** lives in POSIX `sh` helpers under `luci-app-daede/root/usr/share/luci-app-daede/`, invoked from JS via `fs.exec`:
  - `pkg-info.sh <pkg>` — installed version/state of dae|daed|luci-app-daede
  - `update-pkg.sh <pkg>` — in-place package update
  - `update-geo.sh <geoip|geosite>` — geo data update
  - `gen-dae-config.sh <generate|import>` — build/import dae config
  - `proxy-check.sh` — verify dae's transparent proxy actually reaches out

### The ACL is load-bearing — update it whenever the UI calls something new

`luci-app-daede/root/usr/share/rpcd/acl.d/luci-app-daede.json` is a **fine-grained allowlist**: every `fs.exec` command (with its exact args) and every read/written file path is explicitly listed. If JS calls a command or touches a path that is **not** in this file, it fails silently with permission denied — not an obvious error. So any new `fs.exec(...)`, file read, or file write in a view **must** have a matching entry added to `acl.d` (commands are matched literally, including arguments).

### i18n

Translations live in `luci-app-daede/po/zh-cn/daede.po`; `Build/Prepare` runs `po2lmo` to produce the `.lmo`. Add new translatable strings to the `.po`.

## Build & release

Packages are normally built in CI, not locally.

- `.github/workflows/release.yml` builds via `openwrt/gh-action-sdk` (here pinned to the `kenzok8/gh-action-sdk` fork) across a matrix of 8 archs × SDK `24.10` and `25.12`, then publishes to GitHub Releases **and** a Backblaze B2 feed served at `down.dllkids.xyz`. Triggered by `v*` tag push or manual `workflow_dispatch`.
- `luci-app-daede` is `PKGARCH:=all` (noarch); `dae`/`daed` are per-arch Go binaries whose `go.mod` needs Go 1.26 — the SDK's bundled golang is overridden via `GOLANG_REPO` (that's why the fork is used).
- Backend the LuCI app pulls in is a Makefile build-config choice (`daed` by default, or `dae`).

## Bumping the version (required for updates to be seen)

opkg/apk treat the package as current unless the version changes. Any change under `luci-app-daede/htdocs/` or `root/` must bump `luci-app-daede/Makefile` (`PKG_VERSION` for feature-sized changes, `PKG_RELEASE` for small fixes), or installed routers won't pick up the new files and conffiles won't be rebuilt.

## Test deployment

Test router (OpenWrt, **252**): `root@192.168.3.252` on SSH **port 9167**.

After copying changed JS/scripts/ACL onto the router, LuCI caches them — clear and restart so changes take effect (this is what `postinst` does):

```sh
rm -rf /tmp/luci-indexcache* /tmp/luci-modulecache
/etc/init.d/rpcd restart
/etc/init.d/uhttpd restart
```

ACL changes specifically require an `rpcd` restart; new/renamed JS views require clearing the index/module cache.

End-user install (documented in README) is `wget -qO- https://down.dllkids.xyz/openwrt-feed/openwrt-feed-setup.sh | sh` then `apk add dae daed luci-app-daede`.

## CodeGraph

`.codegraph/` is present — use the codegraph tools for exploration ("how does X work", tracing) instead of broad grep/read.
