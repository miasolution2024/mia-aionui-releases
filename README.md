# MIA desktop app — releases

Repo private → download chỉ trong org, auto-update tắt.

## Cài

```bash
xattr -dr com.apple.quarantine /Applications/AionUi.app     # macOS
```
```powershell
Unblock-File -Path .\AionUi-2.2.2-win-x64.exe               # Windows
```

## Build

Windows phải build trên Windows. Cần Node 22, bun, C++ toolchain, Python 3.

```bash
git clone https://github.com/alextran8320/AionUi.git && cd AionUi
bun install
```

```bash
export MIA_CLOUD_URL="https://mia-second-brain.miaai.io.vn"
export ANALYTICS_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/events"
export ANALYTICS_INGEST_KEY="<ANALYTICS_INGEST_KEYS của server>"
export LOGGING_ENDPOINT="https://mia-second-brain.miaai.io.vn/api/logs"
```
```cmd
set "MIA_CLOUD_URL=https://mia-second-brain.miaai.io.vn"
set "ANALYTICS_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/events"
set "ANALYTICS_INGEST_KEY=<ANALYTICS_INGEST_KEYS của server>"
set "LOGGING_ENDPOINT=https://mia-second-brain.miaai.io.vn/api/logs"
```

CMD: nháy bọc cả vế, không bọc riêng giá trị.

```bash
bun run build-mac:arm64
bun run build-win:x64
```

Thấy `⚠️ MIA_CLOUD_URL is not set` thì dừng — app sẽ không có login. File ra ở `out/`.

## Publish

```bash
gh release create v2.2.2 --repo miasolution2024/mia-aionui-releases \
  out/AionUi-2.2.2-mac-arm64.dmg out/AionUi-2.2.2-mac-arm64.zip out/latest-mac.yml
```

`.exe` + `latest.yml` đẩy vào **cùng tag**. Không bỏ `.zip` — updater dùng nó.

## Ký

Chưa ký: Windows cảnh báo vẫn cài được, macOS chặn, macOS auto-update không chạy.
Apple Developer 99 USD/năm, set `CSC_NAME` `APPLE_ID` `APPLE_APP_SPECIFIC_PASSWORD`
`APPLE_TEAM_ID` rồi build.
