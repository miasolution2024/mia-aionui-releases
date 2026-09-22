# MIA desktop app — builds and releases

Installers for the MIA build of AionUi. Source is in a private fork.

**Internal for now.** The repo is private, so download links only work inside the
org and **auto-update is off** — electron-updater cannot read a private repo
without a token. Making it public turns updates back on for Windows; macOS also
needs a Developer ID.

## Install

```bash
# macOS — after dragging to Applications
xattr -dr com.apple.quarantine /Applications/AionUi.app
```

```powershell
# Windows — or just click More info → Run anyway
Unblock-File -Path .\AionUi-2.2.2-win-x64.exe
```

Signing in needs a MIA account.

## Build

One platform per machine. **Windows cannot be built on macOS** — the build runs
`aioncore.exe` to generate managed resources.

Needs Node 22, bun, and a C++ toolchain (Xcode CLT / VS Build Tools 2022
"Desktop development with C++") plus Python 3.

```bash
git clone https://github.com/alextran8320/AionUi.git && cd AionUi
bun install
```

Set these — they are baked in at build time, and `MIA_CLOUD_URL` alone decides
whether the app has a sign-in at all:

```bash
export MIA_CLOUD_URL="https://mia-second-brain.miaai.io.vn"
export ANALYTICS_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/events"
export ANALYTICS_INGEST_KEY="<server's ANALYTICS_INGEST_KEYS>"
export LOGGING_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/logs"
```

```powershell
$env:MIA_CLOUD_URL = "https://mia-second-brain.miaai.io.vn"   # etc.
```

```cmd
set "MIA_CLOUD_URL=https://mia-second-brain.miaai.io.vn"
```

In CMD the quotes wrap the whole assignment — `set VAR="value"` puts the quotes
*inside* the value. Keep the variables and the build in one script; a fresh
terminal loses them silently.

```bash
bun run build-mac:arm64      # macOS
bun run build-win:x64        # Windows
```

The build prints what it baked in. `⚠️ MIA_CLOUD_URL is not set` means stop —
you are building a stock AionUi with no login. Installers land in `out/`,
10–20 minutes on a first run.

`AIONCORE_BOOTSTRAP_SECRET` is not set here: the app generates its own per
machine.

## Publish

```bash
gh release create v2.2.2 --repo miasolution2024/mia-aionui-releases \
  out/AionUi-2.2.2-mac-arm64.dmg out/AionUi-2.2.2-mac-arm64.zip out/latest-mac.yml
```

Tag is `v` + the version in `package.json`. Windows artifacts (`.exe`,
`latest.yml`) go on the **same tag**. The macOS `.zip` is not optional —
`latest-mac.yml` points at it, and the updater swaps the `.app` out of the zip.

## Signing

Needed before customers, skippable for testers. Unsigned: Windows warns but
installs, macOS refuses outright, and **macOS auto-update never works** — it
matches signing identities, and ad-hoc has none.

Apple Developer Program is 99 USD/year. Sign *and* notarize; electron-builder
does both when `CSC_NAME`, `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD` and
`APPLE_TEAM_ID` are set. Verify with `spctl -a -vvv /Applications/AionUi.app`.

## Upgrades

Same `appId`, so a new version replaces the old rather than sitting beside it.
User data survives — nobody signs in again.
