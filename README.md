# MIA desktop app — builds and releases

Installers for the MIA build of AionUi, and how to produce them. The source
lives in a private fork; this repository holds the binaries and the procedure.

**Internal for now.** The repository is private, so the download links only work
for people in the organisation — customers get the installer sent to them. Two
things follow from that, and both are temporary:

- **Auto-update is off.** `electron-updater` cannot read releases from a private
  repository without a token, and a token shipped inside the app is a token
  anyone can extract. Making this repository public is what turns updates on.
- **macOS auto-update needs more than that.** See [Signing](#signing).

---

## Installing

**macOS** — open the `.dmg`, drag AionUi to Applications, then:

```bash
xattr -dr com.apple.quarantine /Applications/AionUi.app
```

Gatekeeper refuses unsigned apps outright, and since macOS 15 the old
Control-click → Open shortcut is gone. That command removes the mark the browser
attaches to downloads, which is what Gatekeeper actually checks.

**Windows** — run the `.exe`. SmartScreen warns about an unknown publisher;
**More info → Run anyway** installs it. Or clear the mark first:

```powershell
Unblock-File -Path .\AionUi-2.2.2-win-x64.exe
```

Signing in needs a MIA account. Without one the app stops at the sign-in screen.

---

## Building

One platform per machine. **A Windows installer cannot be built on macOS**: the
build runs the aioncore binary to generate its managed resources, and
`aioncore.exe` is ENOEXEC there. No flag works around it.

### Prerequisites

| | macOS | Windows |
|---|---|---|
| Node.js | 22.x | 22.x |
| bun | `curl -fsSL https://bun.sh/install \| bash` | `powershell -c "irm bun.sh/install.ps1 \| iex"` |
| Compiler | Xcode Command Line Tools | Visual Studio Build Tools 2022, "Desktop development with C++" |
| Python 3 | preinstalled | required by node-gyp |

### 1. Get the source

```bash
git clone https://github.com/alextran8320/AionUi.git
cd AionUi
bun install
```

### 2. Set the configuration

These are read at **build time** and frozen into the bundle. A double-clicked
app inherits no environment, so whatever is missing here is missing for every
customer. `MIA_CLOUD_URL` alone decides whether the app asks for a sign-in —
without it you have built a stock AionUi that opens straight into the workspace,
and nothing in the build will say so until someone installs it.

`AIONCORE_BOOTSTRAP_SECRET` is deliberately not among them: the app generates
its own per machine on first launch. One baked into the installer would be
shared by every customer.

**macOS / Linux**

```bash
export MIA_CLOUD_URL="https://mia-second-brain.miaai.io.vn"
export ANALYTICS_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/events"
export ANALYTICS_INGEST_KEY="<the server's ANALYTICS_INGEST_KEYS>"
export LOGGING_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/logs"
```

**Windows, PowerShell**

```powershell
$env:MIA_CLOUD_URL        = "https://mia-second-brain.miaai.io.vn"
$env:ANALYTICS_ENDPOINT   = "https://mia-second-brain.miaai.io.vn/api/events"
$env:ANALYTICS_INGEST_KEY = "<the server's ANALYTICS_INGEST_KEYS>"
$env:LOGGING_ENDPOINT     = "https://mia-second-brain.miaai.io.vn/api/logs"
```

**Windows, CMD** — the quotes wrap the whole assignment. Writing
`set VAR="value"` puts the quote marks *inside* the value, and the URL then
matches nothing.

```cmd
set "MIA_CLOUD_URL=https://mia-second-brain.miaai.io.vn"
set "ANALYTICS_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/events"
set "ANALYTICS_INGEST_KEY=<the server's ANALYTICS_INGEST_KEYS>"
set "LOGGING_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/logs"
```

The ingest key is the server's `ANALYTICS_INGEST_KEYS`. It is write-only and
already ships inside every installer, so it is not a secret in the usual sense —
but it is not written here either, because this file is easier to read than a
binary is to disassemble. The read key never leaves the server.

Variables live in one shell. A failed build, a `git pull`, a fresh terminal —
any of those loses them silently. Keep them and the build command in one file:

```bat
@echo off
set "MIA_CLOUD_URL=https://mia-second-brain.miaai.io.vn"
set "ANALYTICS_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/events"
set "ANALYTICS_INGEST_KEY=..."
set "LOGGING_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/logs"

call bun run build-win:x64
```

`call` is not optional. Without it the batch file exits at that line when `bun`
resolves to a `.cmd` shim, and nothing after it runs.

### 3. Build

```bash
bun run build-mac:arm64      # macOS
bun run build-win:x64        # Windows
```

Watch the first lines. The build reports what it baked in:

```
🔧 MIA build configuration
   sign-in   https://mia-second-brain.miaai.io.vn
   analytics https://mia-second-brain.miaai.io.vn/api/events
   logs      https://mia-second-brain.miaai.io.vn/api/logs
   key       set
```

`⚠️  MIA_CLOUD_URL is not set` means the variables did not arrive. Stop there
rather than waiting out the build.

Installers land in `out/`. Expect 10–20 minutes on a first run; later runs reuse
the Electron and aioncore caches.

### 4. Confirm before handing it over

```bash
grep -ro "mia-second-brain.miaai.io.vn\|localhost:4319" out/main out/renderer | sort -u
```

```powershell
Select-String -Path out\main\chunks\*.js,out\renderer\assets\*.js `
  -Pattern "mia-second-brain|localhost:4319" | Select-Object -First 5
```

Only the live host should appear. A `localhost:4319` means step 2 did not take
effect, and the app will work on your machine and nowhere else.

---

## Signing

Skippable for internal testers, not for customers.

| | Unsigned | Signed |
|---|---|---|
| Windows install | SmartScreen warns, installs anyway | Quiet |
| macOS install | Gatekeeper refuses | Quiet |
| macOS auto-update | **Never works** | Works |

That last row surprises people. Squirrel.Mac checks that the incoming version
carries the same signing identity as the running one. An ad-hoc signature has no
identity to match, so updates are refused however correctly the release is
published.

An Apple Developer Program membership (99 USD/year) yields a *Developer ID
Application* certificate. Signing alone is not enough — the app must also be
notarized: uploaded to Apple, scanned, and the returned ticket stapled to it.
electron-builder does all three when these are set:

```bash
export CSC_NAME="Developer ID Application: <name> (TEAMID)"
export APPLE_ID="<apple id>"
export APPLE_APP_SPECIFIC_PASSWORD="<from appleid.apple.com>"
export APPLE_TEAM_ID="<TEAMID>"
```

Check with `spctl -a -vvv /Applications/AionUi.app`, which should answer
`source=Notarized Developer ID`.

Windows signing is a separate purchase from a commercial CA, and since 2023 the
key must live on hardware, which complicates CI.

---

## Publishing

```bash
gh release create v2.2.2 \
  --repo miasolution2024/mia-aionui-releases \
  out/AionUi-2.2.2-mac-arm64.dmg \
  out/AionUi-2.2.2-mac-arm64.zip \
  out/latest-mac.yml
```

The tag is `v` plus the version in `package.json`. Add the Windows artifacts to
the **same tag** — the updater looks up a version, not a platform.

| Platform | Files |
|---|---|
| macOS | `.dmg`, `.zip`, `latest-mac.yml` |
| Windows | `.exe`, `latest.yml` |

The `.zip` is not optional on macOS: `latest-mac.yml` points `path:` at it, and
the updater swaps the `.app` out of the zip. The `.dmg` is only for a first
install.

The `latest-*.yml` files matter once this repository is public and updates are
on. Until then they are harmless to include and one less thing to remember.

---

## Upgrades

Installing a newer version replaces the old one rather than sitting beside it —
same `appId`, so NSIS removes the previous install first and macOS overwrites the
bundle. User data survives, including `mia-auth.json` and `mia-session.json`, so
nobody signs in again after an upgrade.
