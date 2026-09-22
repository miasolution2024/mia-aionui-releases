# MIA desktop app — releases

Private repo → downloads work inside the org only, auto-update is off.

## Install

```bash
# macOS — after dragging to Applications
xattr -dr com.apple.quarantine /Applications/AionUi.app
```
```powershell
# Windows — or click More info → Run anyway
Unblock-File -Path .\AionUi-2.2.2-win-x64.exe
```

## Build — common

Windows must be built on Windows. Node 22, bun, Python 3.

```bash
git clone https://github.com/alextran8320/AionUi.git && cd AionUi
bun install
```

Stop if the build prints `⚠️ MIA_CLOUD_URL is not set` — the app will have no
login. Installers land in `out/`.

## Build — macOS

Needs Xcode Command Line Tools: `xcode-select --install`

```bash
export MIA_CLOUD_URL="https://mia-second-brain.miaai.io.vn"
export ANALYTICS_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/events"
export ANALYTICS_INGEST_KEY="<server's ANALYTICS_INGEST_KEYS>"
export LOGGING_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/logs"

bun run build-mac:arm64     # Apple Silicon
bun run build-mac:x64       # Intel
```

## Build — Windows

Needs VS Build Tools 2022, "Desktop development with C++".

```cmd
set "MIA_CLOUD_URL=https://mia-second-brain.miaai.io.vn"
set "ANALYTICS_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/events"
set "ANALYTICS_INGEST_KEY=<server's ANALYTICS_INGEST_KEYS>"
set "LOGGING_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/logs"

bun run build-win:x64
```

Quotes wrap the whole assignment — `set VAR="value"` puts them inside the value.
Put the vars and the build in one `.bat`; a new terminal loses them.

## Release

Bump first — artifact names and `latest*.yml` both read this version.

```bash
npm version 2.2.3 --no-git-tag-version   # root package.json
```

Build (above), then publish on a tag matching the version:

```bash
gh release create v2.2.3 --repo miasolution2024/mia-aionui-releases \
  out/AionUi-2.2.3-mac-arm64.dmg out/AionUi-2.2.3-mac-arm64.zip out/latest-mac.yml
```

`.exe` + `latest.yml` go on the **same tag**. Keep the `.zip` — the updater uses it.
Leave `packages/desktop/package.json` at `0.0.0`; it is not read.

## Signing

Unsigned: Windows warns but installs, macOS refuses, macOS auto-update never works.

Apple Developer is 99 USD/year. Set `CSC_NAME`, `APPLE_ID`,
`APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID`, then build.
