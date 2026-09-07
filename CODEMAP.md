# CODEMAP — fjolskylduoryggisverndar/ninjashield-ci

> File-by-file map of the NinjaShield CI repo: five GitHub Actions workflows (one orchestrator, three reusable per-platform builders, one daily MSIX mirror), plus LICENSE and a leftover .gitignore. Identical to `BuddhaJumpApp/buddhajump-ci` except for the two pieces listed at the end. Every claim cites `file:line`.

写于 2026-09-07。行号以 `kamevpn-ci`/`88888vpn-ci` 之外的 fork 为准；那两个仓库的 `build-android.yml` 和 `build-windows.yml` 第 1 行多一条注释，行号整体 +1。总览见 [README.md](README.md)，全估算仓库地图见 hydra 的 [docs/REPOS.md](https://github.com/fjolskylduoryggisverndar/hydra/blob/HEAD/docs/REPOS.md)。

```
.
├── .github/workflows/
│   ├── build.yml            91 行  编排：读版本 → 并行调三个平台 workflow
│   ├── build-android.yml   148 行  AAB+APK；AAB→Play internal；APK→Release(latest)
│   ├── build-apple.yml     208 行  iOS ipa + macOS zip/pkg；可选上传 ASC（无 DMG 段）
│   ├── build-windows.yml   213 行  便携 zip(+VC++ DLL)；可选 MSIX→Microsoft Store
│   └── publish.yml          36 行  每日 cron：把 Store 上的 MSIX 镜像回 Release
├── .gitignore                Rust 模板残留，与本仓库无关
└── LICENSE                   GNU GPL v3 全文
```

没有脚本、没有源码。所有构建输入都来自 `PRIVATE_REPO`（应为 `fjolskylduoryggisverndar/ninjashield`）指向的私有源码仓库。

---

## `.github/workflows/build.yml`

- **角色**：唯一的人工入口；解析版本并扇出到三个可复用 workflow。
- **触发**（`:10-34`）：`repository_dispatch: [pubspec-updated]`（无可用发送方，见 README §2）；`workflow_dispatch`，4 个布尔输入 `publish_to_stores` / `publish_ios_testflight` / `publish_macos_testflight` / `build_windows_store_package`，默认全 false。
- **全局**：`env.GITHUB_TOKEN`、`env.PRIVATE_REPO`（`:3-5`）；`permissions: contents: write`（`:7-8`）。
- **job `check-for-version`**（ubuntu-latest，`:37-59`）
  - `actions/checkout@v7` 检出 `${{ env.PRIVATE_REPO }}`，token `secrets.ORG_TOKEN`（`:44-48`）。
  - step `get-info`（`:50-59`）：`NAME` ← pubspec `^name:`（本品牌为 `ninjashield`）；`PKG` ← `android/app/build.gradle.kts` 的 `applicationId`；`VERSION` ← pubspec `^version:` 整行加 `v`（**保留 `+N`**）。
  - outputs：`name` / `version` / `pkg`（`:39-42`）。
- **job `build-windows` / `build-android` / `build-apple`**（`:61-91`）：`uses: ./.github/workflows/build-*.yml`，`secrets: inherit`；布尔输入包成 `github.event_name == 'workflow_dispatch' && inputs.X || false`（`:68-69, 88-90`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`。
- **外部联系**：私有仓库 `fjolskylduoryggisverndar/ninjashield`（读 `pubspec.yaml`、`android/app/build.gradle.kts`）。

---

## `.github/workflows/build-android.yml`

- **角色**：Android 构建 + Play 上传 + APK 发布，并**决定 GitHub Release 的 latest 指针**。
- **触发**（`:10-24`）：`workflow_call`，输入 `name` / `pkg` / `version`。
- **job `build-android-app`**（ubuntu-latest，`:27-148`）
  1. checkout 私有仓库（`:30-34`）。
  2. `actions/setup-java@v5` temurin 17（`:36-40`）；`subosito/flutter-action@v2` `channel: stable`（`:42-46`）；`actions/cache@v5` pub 缓存（`:48-56`）。
  3. step `build-env`（`:58-68`）：去 `v`、去 `+N`，`BUILD_NUMBER=a*100+b*10+c`，`sed -i` 回写 pubspec。
  4. `flutter pub get`、`flutter --version`（`:70-76`）。
  5. 签名（`:78-86`）：`PKCS12_BASE64` → `android/app/<PKCS12_NAME>-keystore.jks`；heredoc 写 `android/key.properties`。
  6. 构建（`:88-110`）：`rm -rf .dart_tool android/.gradle`；`flutter gen-l10n`；`flutter pub run flutter_launcher_icons`；`flutter build appbundle --release --obfuscate --split-debug-info=debug-info/android --suppress-analytics --no-tree-shake-icons` → `<name>-android-<version>.aab`；`flutter build apk` → `<name>-android-universal-<version>.apk`。
  7. 解码 `GOOGLE_SERVICE_ACCOUNT_BASE64`（`:112-113`）。
  8. step `publish`：`r0adkll/upload-google-play@v1`，`track: internal`，`status: completed`，`continue-on-error: true`（`:115-128`）。**无 `if:`，每次都跑。**
  9. `softprops/action-gh-release@v3` 传 APK，`make_latest: true`（`:130-138`）。
  10. `if: steps.publish.outcome == 'failure'` 时再传 AAB（`:140-148`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PKCS12_BASE64`、`PKCS12_NAME`、`PKCS12_PASSWORD`、`GOOGLE_SERVICE_ACCOUNT_BASE64`。
- **外部联系**：Google Play internal（包名 = `inputs.pkg`）；本仓库 Release；`dl.ninjashield.xyz` Worker 通过 `/releases/latest` 间接依赖第 9 步。

---

## `.github/workflows/build-apple.yml`

- **角色**：iOS + macOS 构建、可选 ASC/TestFlight 上传。**没有母版的 Developer ID 公证 DMG 段。**
- **触发**（`:10-39`）：`workflow_call`，输入 `name` / `pkg` / `version` + 布尔 `publish_to_stores` / `publish_ios_testflight` / `publish_macos_testflight`。
- **job `build-apple-app`**（macos-latest，matrix `[ios, macos]`，`fail-fast: false`，`:42-208`）
  1. checkout（`:49-53`）；Flutter stable（`:55-59`）；`maxim-lobanov/setup-xcode@v1` `latest-stable`（`:61-64`）；pub 缓存（`:66-74`）。
  2. build env（`:76-94`）：`a*100+b*10+c`，BSD `sed -i ''` 回写 pubspec。
  3. `flutter pub get`、`flutter --version`（`:95-101`）。
  4. keychain（`:103-125`）：导入 `APPLE_PKCS12_BASE64`；若 `APPLE_INSTALLER_PKCS12_BASE64` 非空再导入 installer 证书（`:119-123`）。
  5. 描述文件（`:127-131`）：`APPLE_PROVISIONING_BASE64` → `base64 -d | tar -xJf -`。
  6. 构建（`:133-168`）：`flutter gen-l10n`；`flutter_launcher_icons`；ios → `flutter build ipa --export-options-plist=ios/ExportOptions.plist` → `<name>-<version>.ipa`（`:141-149`）；macos → `flutter build macos`，按开关 `productbuild --sign` 出 `.pkg` 或 `ditto` 出 `<name>-macos-<version>.zip`（`:150-167`）。
  7. Clean up keychain（`if: always()`，`:170-174`）。
  8. Setup ASC API key（条件 `publish_to_stores || (ios_tf && ios) || (macos_tf && macos)`，`:176-181`）。
  9. step `publish`：`xcrun altool --upload-app --type ios|osx`，`continue-on-error`（`:183-193`）。
  10. Upload to GitHub release（`if: !inputs.publish_to_stores || steps.publish.outcome == 'failure'`，`overwrite_files: true`，`:195-204`）。
  11. Clean up API key（`:206-208`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`APPLE_PKCS12_BASE64`、`APPLE_PKCS12_PASSWORD`、`APPLE_INSTALLER_PKCS12_BASE64`、`APPLE_INSTALLER_PKCS12_PASSWORD`、`APPLE_PROVISIONING_BASE64`、`APPLE_API_KEY_BASE64`、`APPLE_API_KEY_ID`、`APPLE_API_ISSUER_ID`。
- **外部联系**：App Store Connect / TestFlight；本仓库 Release；私有仓库的 `ios/ExportOptions.plist`（`method: app-store`，`signingStyle: manual`，里面的 bundle id 是配 `APPLE_PROVISIONING_BASE64` 的依据）。

---

## `.github/workflows/build-windows.yml`

- **角色**：Windows 便携 zip（含 VC++ 运行库）+ 可选 Microsoft Store MSIX。
- **触发**（`:11-35`）：`workflow_call`，输入 `name` / `pkg` / `version` + 布尔 `build_store_package` / `publish_to_stores`。
- **job `build-windows-app`**（windows-latest，`outputs.submission-success`（`:40-41`，无人消费），`:38-213`）
  1. checkout（`:43-47`）。
  2. Store 前置（条件 `build_store_package || publish_to_stores`）：`microsoft/microsoft-store-apppublisher@v1.3`（`:49-51`）；`msstore reconfigure` 用 `PARTNER_TENANT/SELLER/CLIENT/SECRET`（`:52-54`）；`Install-Module StoreBroker`（`:56-59`，未用）。
  3. `rm -rf build windows/.Dart_tool .dart_tool`（`:61-63`）。
  4. Flutter stable（`:65-69`）；pub 缓存（`:71-79`）；`flutter config --enable-windows-desktop --enable-native-assets`（`:81-85`）；`flutter pub get`（`:88-92`）。
  5. `flutter build windows --release --obfuscate --split-debug-info=debug-info/windows-amd64 ...`（`:96-107`）。**没有 gen-l10n、没有 launcher_icons。**
  6. Bundle VC++ runtime（`:109-139`）：`msvcp140.dll`、`vcruntime140.dll`、`vcruntime140_1.dll` 经 `vswhere` 定位后拷到 `build/windows/x64/runner/Release`，缺一个就 `throw`。
  7. `Compress-Archive` → `<name>-windows-amd64-<version>.zip`（`:141-146`）；上传 Release，`overwrite_files: true`（`:148-156`）。
  8. Store 路径（条件同 2）：`MSIX_VERSION=a.b.c.0`、`MSIX_FILENAME=<name>-windows-amd64-<version>-store.msix`（`:158-170`）；`dart run msix:create ... --store true --sign-msix false`（`:172-189`）；`msstore publish -id PARTNER_STORE`，`continue-on-error`（`:198-202`）；失败则 MSIX 上传 Release（`:204-213`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PARTNER_TENANT`、`PARTNER_SELLER`、`PARTNER_CLIENT`、`PARTNER_SECRET`、`PARTNER_APP`、`PARTNER_COMPANY`、`PARTNER_CN`、`PARTNER_STORE`。
- **外部联系**：Microsoft Partner Center / Store；本仓库 Release；私有仓库的 `assets/logo.png`。

---

## `.github/workflows/publish.yml`

- **角色**：把 Microsoft Store 上已上架的 MSIX 镜像回本仓库 Release。
- **触发**（`:3-11`）：`schedule: '0 0 * * *'`；`workflow_dispatch`；`push` 到 **`master`** 且路径为 `.github/workflows/publish.yml`。本仓库默认分支为 `master`——若不是 `master`，push 触发永远不命中。
- **job `upload-to-release`**（ubuntu-24.04，`:21-36`）：checkout 私有仓库（`:24-28`，未用）；`JasonWei512/Upload-Microsoft-Store-MSIX-Package-to-GitHub-Release@v1`，`store-id: PARTNER_STORE`，`asset-name-pattern: <PKCS12_NAME>_{version}_{arch}`，`continue-on-error: true`（`:30-36`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PARTNER_STORE`、`PKCS12_NAME`。

---

## `.gitignore` / `LICENSE`

`.gitignore`：`.DS_Store`、`.claude`、`CLAUDE.md`、`debug/`、`target/`、`Cargo.lock`、`**/*.rs.bk`、`*.pdb`——Rust 模板残留；注意它忽略了 `CLAUDE.md`。`LICENSE`：GNU GPL v3 全文（`LICENSE:1-2`），只覆盖 workflow 文件。

---

## 与母版的差异

| 母版有、本仓库没有 | 母版位置 | 影响 |
|---|---|---|
| `build-apple.yml` 的「Build notarized DMG (macOS direct distribution)」+「Upload DMG to GitHub release」 | 母版 `build-apple.yml:170-306` | 本仓库 macOS 只有 `.zip`/`.pkg`，没有可直接下载安装的 DMG。移植时要改母版写死的 `BuddhaJump_DevID_macOS.provisionprofile` / `BuddhaJump_Service_DevID_macOS.provisionprofile`（母版 `:228, 230`），并新增 `APPLE_DEVELOPER_ID_PKCS12_BASE64` / `_PASSWORD`、`APPLE_DEVID_PROVISIONING_BASE64` 三个 secret。 |
| `build-android-hotfix.yml` | 母版整个文件（120 行） | 本仓库不能出「只 APK、不进 Play、不动 latest」的侧车构建。文件品牌无关，可直接拷贝。 |

其余 5 个文件与母版逐字节相同（`kamevpn-ci`/`88888vpn-ci` 的 `build-android.yml`、`build-windows.yml` 多一行头注释）。

## 跨文件的共同模式

| 模式 | 出处 |
|---|---|
| 每个 workflow 顶部 `env.GITHUB_TOKEN` + `env.PRIVATE_REPO`，`permissions: contents: write` | 5 个文件开头 |
| 第一步永远是 `actions/checkout@v7` + `repository: env.PRIVATE_REPO` + `token: secrets.ORG_TOKEN` | 每个 job 第一步 |
| Release 全部通过 `softprops/action-gh-release@v3`，`tag_name` = `inputs.version`，`name: Release <tag>` | 6 处 |
| 商店上传步骤 `continue-on-error: true` + `id: publish`，失败时把产物改传 Release | build-android `:115-148`、build-apple `:183-204`、build-windows `:198-213` |
| build number `a*100+b*10+c` 回写 pubspec | build-android `:58-68`、build-apple `:76-94` |
| `flutter gen-l10n` + `flutter_launcher_icons` 构建前现跑 | build-android `:93-94`、build-apple `:138-139`；**build-windows 没有** |
| Flutter `channel: stable`、Xcode `latest-stable`，均未锁版本 | 所有构建 job |

## 第三方 action 一览

| action | 版本 | 用处 |
|---|---|---|
| `actions/checkout` | v7 | 检出私有仓库 |
| `actions/setup-java` | v5 | Temurin 17 |
| `subosito/flutter-action` | v2 | Flutter stable |
| `actions/cache` | v5 | pub 缓存 |
| `maxim-lobanov/setup-xcode` | v1 | Xcode latest-stable |
| `r0adkll/upload-google-play` | v1 | AAB → Play internal |
| `softprops/action-gh-release` | v3 | 所有 Release 上传 |
| `microsoft/microsoft-store-apppublisher` | v1.3 | `msstore` CLI |
| `JasonWei512/Upload-Microsoft-Store-MSIX-Package-to-GitHub-Release` | v1 | publish.yml |
