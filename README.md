# fjolskylduoryggisverndar/ninjashield-ci

> GitHub Actions pipeline for the NinjaShield Flutter VPN client: checks out the private app repo `fjolskylduoryggisverndar/ninjashield`, builds Android/iOS/macOS/Windows, publishes the download assets as GitHub Releases **on this repo**, and optionally pushes to Play internal / App Store Connect / TestFlight / Microsoft Store. A byte-for-byte fork of `BuddhaJumpApp/buddhajump-ci` minus two pieces (see §10).

本文件写于 2026-09-07，每一条事实都标注了在 `.github/workflows/*.yml` 里的出处（`文件:行号`，行号以 `kamevpn-ci`/`88888vpn-ci` 之外的 fork 为准；那两个仓库的 `build-android.yml` 和 `build-windows.yml` 多一行头注释，行号整体 +1）。
没有标注出处、来自运维记忆的内容，全部集中在文末「来自运维记忆（未在代码中复核）」一节。
逐文件细节见 [CODEMAP.md](CODEMAP.md)。全部 42 个仓库的地图见 hydra 仓库的 [docs/REPOS.md](https://github.com/fjolskylduoryggisverndar/hydra/blob/HEAD/docs/REPOS.md)。

---

## 1. 这个仓库是什么、不是什么

| | |
|---|---|
| **是** | NinjaShield 客户端的构建流水线。仓库里只有 5 个 workflow 文件、LICENSE 和一个 `.gitignore`，**没有任何应用源码**。 |
| **是** | 客户端下载产物的发布地：每次构建用 `softprops/action-gh-release` 在**本仓库**建 Release（`build-android.yml:130-138`、`build-windows.yml:148-156`、`build-apple.yml:195-204`）。官网 `ninjashield.xyz` 的下载按钮最终指向这里（§7）。 |
| **是** | `BuddhaJumpApp/buddhajump-ci`（母版）的 fork。母版改了 workflow，要手工把同名文件拷过来（§10）。 |
| **不是** | 源码仓库。源码在私有仓库 `fjolskylduoryggisverndar/ninjashield`，由 workflow 通过 `PRIVATE_REPO` + `ORG_TOKEN` 检出（`build.yml:44-48`）。改代码、改版本号都要去那边。 |
| **不是** | 自动化的。虽然 `on:` 里写了 `repository_dispatch`，但实际没有可用的发送方（§2），每一次发版都是人手点 `workflow_dispatch`。 |

三件套：`fjolskylduoryggisverndar/ninjashield`（源码，私有）→ **`fjolskylduoryggisverndar/ninjashield-ci`（本仓库，公开）** → `fjolskylduoryggisverndar/ninjashield-web`（官网快照，域名 `ninjashield.xyz`）。
本仓库默认分支：`master`。当前最新 Release tag：`v1.2.6+126`。
`.gitignore` 是 Rust 项目模板残留（`target/`、`Cargo.lock`、`*.rs.bk`），与本仓库内容无关。

---

## 2. 触发方式：写的和真的不一样

`build.yml:10-34` 声明了两个触发器：

```yaml
on:
  repository_dispatch:
    types: [pubspec-updated]
  workflow_dispatch:
    inputs: publish_to_stores / publish_ios_testflight / publish_macos_testflight / build_windows_store_package  # 4 个布尔，默认 false
```

**`repository_dispatch: pubspec-updated` 是死路。**

- 发送方必须向 `api.github.com/repos/fjolskylduoryggisverndar/ninjashield-ci/dispatches` POST `event_type: pubspec-updated`。
- 全估算里只有 `00000vpn`/`88888vpn` 两个源码仓库有会发这个事件的 `.github/workflows/ci.yml`（`ci.yml:58-63`，目标 `secrets.PUBLIC_REPO`），但它们的触发条件是 `push: branches: [master]`（`ci.yml:34-36`），而这两个仓库的默认分支是 `main`——push 永远打不中。其余源码仓库连 `.github/workflows/` 都没有。
- 即便事件真的来了，`build.yml:68-69, 88-90` 把所有布尔输入包在 `github.event_name == 'workflow_dispatch' && inputs.xxx || false` 里，dispatch 触发的构建等价于「四个开关全关」。

**真正的入口是 `workflow_dispatch`，四个布尔输入的实际效果：**

| 输入 | 传给谁 | 实际行为 |
|---|---|---|
| `publish_to_stores` | build-windows、build-apple | Apple：iOS `.ipa` + macOS 签名 `.pkg` 都 `altool --upload-app` 到 App Store Connect（`build-apple.yml:183-193`），且**不再**把 ipa/pkg 传到 GitHub Release，除非上传失败（`build-apple.yml:196`）。Windows：构建 MSIX 并 `msstore publish`（`build-windows.yml:172-202`）。 |
| `publish_ios_testflight` | build-apple | 只把 iOS `.ipa` 上传 ASC（`build-apple.yml:177, 184`）。因为 `publish_to_stores` 为 false，ipa **同时**还会传到 GitHub Release。 |
| `publish_macos_testflight` | build-apple | 用 `productbuild` 签 `.pkg`（需要 Mac Installer Distribution 证书，`build-apple.yml:157-162`）并上传 ASC。缺 `APPLE_INSTALLER_PKCS12_*` 时 `test -n "$INSTALLER_CERT"` 直接失败（`:160`）。 |
| `build_windows_store_package` | build-windows | 名字说「只构建不发布」，但 `Publish to Store` 的条件是 `inputs.build_store_package \|\| inputs.publish_to_stores`（`build-windows.yml:199`）——**它会尝试发布**，失败了才把 MSIX 传到 Release（`:204-213`）。 |

**四个开关全关时（默认）依然会发生的事：**

- Android AAB **无条件**上传 Google Play `internal` 轨道（`build-android.yml:115-128`，没有 `if:`，只有 `continue-on-error`）。
- APK、Windows zip、iOS ipa、macOS zip 全部发到本仓库 GitHub Release，APK 那一步 `make_latest: true`（`build-android.yml:137`）——这一步决定了 dl.* 下载链路指向哪个版本（§7）。

---

## 3. 版本号、build number、tag、资产名

| 名称 | 来源 | 例（pubspec `version: 1.2.9+129`） |
|---|---|---|
| Release tag / `inputs.version` | `build.yml:55-58`：pubspec `version:` 整行加 `v` 前缀，**不去掉 `+N`** | `v1.2.9+129` |
| Android `versionCode` / Apple `CFBundleVersion` | `build-android.yml:62-67`、`build-apple.yml:88-93`：去掉 `v` 和 `+N` 后 `a*100 + b*10 + c`，再回写 pubspec | `129` |
| Windows MSIX 版本 | `build-windows.yml:163-167`：`a.b.c.0` | `1.2.9.0` |

**tag 里的 `+N` 与安装包里的 build number 可以不一致**，tag 只是 pubspec 那一刻的文字。商店看到的永远是算出来的那个。

资产命名（`name` = pubspec `name:` = `ninjashield`；`version` = 上面的 tag）：

| 平台 | 文件名 | 出处 |
|---|---|---|
| Android APK（universal） | `ninjashield-android-universal-<tag>.apk` | `build-android.yml:110` |
| Android AAB（仅 Play 上传失败时） | `ninjashield-android-<tag>.aab` | `build-android.yml:102, 140-148` |
| Windows 便携 zip | `ninjashield-windows-amd64-<tag>.zip` | `build-windows.yml:144` |
| Windows Store MSIX（仅失败回落） | `ninjashield-windows-amd64-<tag>-store.msix` | `build-windows.yml:168` |
| iOS | `ninjashield-<tag>.ipa` | `build-apple.yml:148` |
| macOS 未签 pkg 时 | `ninjashield-macos-<tag>.zip` | `build-apple.yml:164` |
| macOS TestFlight/Store | `ninjashield-<tag>.pkg` | `build-apple.yml:158` |

`ninjashield` 不一定等于品牌名：00000vpn/88888vpn 的 pubspec `name:` 是 `vpn00000`/`vpn88888`（Dart 包名不能以数字开头），dl.* Worker 的 `prefix` 字段也照此写（`app-downloads.js:59-66`）。

---

## 4. 一次 `build.yml` 运行会做什么

`build.yml` 只有一个真正的 job `check-for-version`（ubuntu-latest），检出私有仓库、从 `pubspec.yaml` 读 `name`/`version`、从 `android/app/build.gradle.kts` 读 `applicationId`（`build.yml:50-59`），然后**并行**调用三个可复用 workflow（`build.yml:61-91`，`secrets: inherit`）。

### 4.1 build-android.yml → `build-android-app`（ubuntu-latest）

1. 检出私有仓库（`:30-34`），Temurin JDK 17（`:36-40`），Flutter `stable`（`:42-46`，**未锁定版本**），pub 缓存（`:48-56`）。
2. 推导 build number 并回写 pubspec（`:58-68`）。
3. 写入签名：`PKCS12_BASE64` → `android/app/<PKCS12_NAME>-keystore.jks`；`android/key.properties` 里 storePassword = keyPassword = `PKCS12_PASSWORD`，keyAlias = `PKCS12_NAME`（`:78-86`）。
4. `rm -rf .dart_tool android/.gradle` → `flutter gen-l10n` → `flutter pub run flutter_launcher_icons` → `flutter build appbundle` + `flutter build apk`，都带 `--obfuscate --split-debug-info`（`:88-110`）。
5. AAB → Google Play **internal** 轨道，`status: completed`，`continue-on-error`（`:112-128`）。
6. APK → 本仓库 Release，`make_latest: true`（`:130-138`）。
7. 仅当第 5 步失败：AAB 也传 Release（`:140-148`）。

### 4.2 build-windows.yml → `build-windows-app`（windows-latest）

1. 检出私有仓库（`:43-47`）。仅在 Store 开关打开时装 `msstore` CLI 并 `reconfigure`（`:49-54`）、装 StoreBroker（`:56-59`，后面没用到）。
2. Flutter stable，开启 `windows-desktop` 和 `native-assets`（`:65-85`）。
3. `flutter build windows --release --obfuscate ...`（`:96-107`）。**这里没有 `flutter gen-l10n`、没有 `flutter_launcher_icons`**——Windows 构建直接使用源码仓库里已提交的生成物。
4. **把 VC++ 运行库三个 DLL 拷到 exe 旁边**（`:109-139`）。注释写明：开发机全局有这些 DLL 所以本地永远测不出。
5. `Compress-Archive` 成便携 zip → Release（`:141-156`）。
6. 仅在 Store 开关打开时：算 MSIX 版本、`dart run msix:create --store true --sign-msix false`、`msstore publish -id PARTNER_STORE`（`continue-on-error`），失败则 MSIX 传 Release（`:158-213`）。

### 4.3 build-apple.yml → `build-apple-app`（macos-latest，matrix `ios`/`macos`，`fail-fast: false`）

1. 检出私有仓库、Flutter stable、`setup-xcode latest-stable`（`:49-64`）。
2. 推导 build number 并用 BSD `sed -i ''` 回写 pubspec（`:76-94`）。
3. 建临时 keychain，导入 `APPLE_PKCS12_*`；若设置了 `APPLE_INSTALLER_PKCS12_*` 再导入 Mac Installer Distribution 证书（`:103-125`）。
4. `APPLE_PROVISIONING_BASE64` 是 **base64 编码的 tar.xz**，解到 `~/Library/MobileDevice/Provisioning Profiles`（`:127-131`）。
5. `flutter gen-l10n` + `flutter_launcher_icons` 后：iOS `flutter build ipa --export-options-plist=ios/ExportOptions.plist`；macOS `flutter build macos`，按开关决定 `productbuild` 签 `.pkg` 还是 `ditto` 打 `.zip`（`:133-168`）。
6. 清 keychain 和 profile（`:170-174`，`if: always()`）。
7. 按开关：写 ASC API key（`:176-181`），`xcrun altool --upload-app --type ios|osx`（`:183-193`，`continue-on-error`），按 §2 表里的条件传 Release（`:195-204`），删 key（`:206-208`）。

**本仓库没有母版的「Build notarized DMG」段**（§10）：macOS 只会得到 `.zip`（App Store 签名，Gatekeeper 在商店外不认）或 `.pkg`（商店提交格式，用户装不了）。要给 macOS 用户直接下载的东西，得先从母版移植那一段并准备 Developer ID 证书。

### 4.4 谁跑在哪、产物去哪（速查）

| job | runner | 产物 | 去向 |
|---|---|---|---|
| check-for-version | ubuntu-latest | `name` / `pkg` / `version` | 下游三个 job |
| build-android-app | ubuntu-latest | AAB、APK | AAB → Play internal（总是）；APK → Release（latest）；AAB → Release（仅 Play 失败） |
| build-windows-app | windows-latest | zip；可选 MSIX | zip → Release；MSIX → Microsoft Store（开关）/ Release（发布失败） |
| build-apple-app [ios] | macos-latest | ipa | ASC/TestFlight（开关）；Release（非 `publish_to_stores` 或上传失败） |
| build-apple-app [macos] | macos-latest | zip 或 pkg | 同上 |
| upload-to-release（publish.yml） | ubuntu-24.04 | 从 Microsoft Store 拉回的 MSIX | Release |

---

## 5. 实际用到的 secrets（23 个，全部 `grep secrets\.` 得出）

旧版 README 写的 `ANDROID_KEYSTORE_BASE64 / ANDROID_KEYSTORE_PASSWORD / ANDROID_KEY_PASSWORD / ANDROID_KEY_ALIAS` **在任何 yml 里都不存在**，不要照着配。旧 README 里 `PRIVATE_REPO` 的值也已过期（写的是 `BuddhaJumpApp/ninjashield`），2026-09-07 起源码仓库在 `fjolskylduoryggisverndar/ninjashield`。

| 组 | secret | 用途 | 出处 |
|---|---|---|---|
| 检出 | `PRIVATE_REPO` | 源码仓库 `owner/name`，本仓库应为 `fjolskylduoryggisverndar/ninjashield` | 每个 workflow 的 `env:` + `actions/checkout with.repository` |
| 检出 | `ORG_TOKEN` | 能读该私有仓库的 token | `build.yml:48` 等 5 处 |
| 检出 | `GITHUB_TOKEN` | Actions 自带，`permissions: contents: write`，建 Release 用 | 每个 workflow 的 `env:` |
| Android | `PKCS12_BASE64` | keystore（jks）base64 | `build-android.yml:80` |
| Android | `PKCS12_NAME` | key alias 兼 keystore 文件名前缀；**publish.yml 还拿它当 MSIX 资产名前缀** | `build-android.yml:80-85`、`publish.yml:36` |
| Android | `PKCS12_PASSWORD` | store 密码 = key 密码 | `build-android.yml:82-83` |
| Android | `GOOGLE_SERVICE_ACCOUNT_BASE64` | Play 服务账号 JSON | `build-android.yml:113` |
| Apple | `APPLE_PKCS12_BASE64` / `APPLE_PKCS12_PASSWORD` | App 签名证书 | `build-apple.yml:111-112` |
| Apple | `APPLE_PROVISIONING_BASE64` | App Store 描述文件，tar.xz 再 base64 | `build-apple.yml:131` |
| Apple | `APPLE_INSTALLER_PKCS12_BASE64` / `_PASSWORD` | Mac Installer Distribution 证书（可选；macOS TestFlight/Store 必需） | `build-apple.yml:119-123` |
| Apple | `APPLE_API_KEY_BASE64` / `APPLE_API_KEY_ID` / `APPLE_API_ISSUER_ID` | ASC API key（altool 上传） | `build-apple.yml:180-193` |
| Microsoft | `PARTNER_TENANT` / `PARTNER_SELLER` / `PARTNER_CLIENT` / `PARTNER_SECRET` | `msstore reconfigure` | `build-windows.yml:54` |
| Microsoft | `PARTNER_APP` / `PARTNER_COMPANY` / `PARTNER_CN` | MSIX identity / 发布者显示名 / publisher CN | `build-windows.yml:177-186` |
| Microsoft | `PARTNER_STORE` | Store 应用 ID | `build-windows.yml:202`、`publish.yml:34` |

母版另有 3 个 Developer ID 相关 secret（`APPLE_DEVELOPER_ID_PKCS12_BASE64` / `_PASSWORD`、`APPLE_DEVID_PROVISIONING_BASE64`），本仓库没有引用它们的步骤。

`PRIVATE_REPO + ORG_TOKEN` 机制：本仓库公开、源码仓库私有，所有 job 第一步都用 `ORG_TOKEN` 检出 `PRIVATE_REPO`。源码仓库改名或转移后要同步改 `PRIVATE_REPO`（GitHub 对旧名有重定向，但不要依赖它）。运维记忆说 `ORG_TOKEN` 是 classic PAT——代码只能看出它是个能读私有仓库的 token。

---

## 6. 怎么手动跑一次

```bash
# 普通发版：四平台构建，产物全部进本仓库 Release，AAB 自动进 Play internal
gh workflow run build.yml --repo fjolskylduoryggisverndar/ninjashield-ci --ref master

# 只把 iOS 推到 TestFlight（ipa 同时也会进 Release）
gh workflow run build.yml --repo fjolskylduoryggisverndar/ninjashield-ci --ref master \
  -f publish_ios_testflight=true

# 正式提交商店
gh workflow run build.yml --repo fjolskylduoryggisverndar/ninjashield-ci --ref master \
  -f publish_to_stores=true

# 看进度
gh run list --repo fjolskylduoryggisverndar/ninjashield-ci --workflow build.yml --limit 5
gh run watch --repo fjolskylduoryggisverndar/ninjashield-ci <run-id>
```

`--repo` 必须是 **ci 仓库**而不是源码仓库。运行前先确认 `fjolskylduoryggisverndar/ninjashield` 的 `pubspec.yaml` `version:` 已经改成新版本，否则会往**同一个 tag** 再传一遍（Windows/Apple 那几步 `overwrite_files: true` 会覆盖，Android 那步没有 `overwrite_files`）。

本仓库**没有** `build-android-hotfix.yml`；需要只出 APK、不进 Play、不动 latest 的侧车构建时，先从母版拷那个文件过来。

---

## 7. 下载链路对本仓库的依赖

官网 → `dl.ninjashield.xyz/android|windows` → Cloudflare Worker `fjolsky-downloads` → 302 → `edgeone.gh-proxy.org/https://github.com/fjolskylduoryggisverndar/ninjashield-ci/releases/download/<tag>/<asset>`。

Worker 源码 `XTPU/cloudflare/workers/app-downloads.js`：

- `TABLE["ninjashield.xyz"]` 一行写着 `repo`、`prefix`、`pin`（`app-downloads.js:22-68`）。**`repo` 必须与本仓库当前的 `owner/name` 一致**。
- 请求时对 `https://github.com/<repo>/releases/latest` 发 `redirect: "manual"` 的 fetch，从 `Location` 头解析 tag，边缘缓存 600 秒（`:81-111`）。
- 任何失败（仓库改名、转私有、超时）都**静默回落到 `pin`**——用户会拿到一个很旧但仍存在的版本，而不是报错（`:105-110, 137, 156`）。
- 资产名由 `prefix` 和 tag 拼出（`:76-79`），必须与 §3 一致。

规则：改本仓库名字/归属/可见性之前，先改 Worker 的 `TABLE` 并重新部署；`latest` 由 `build-android.yml:137` 的 `make_latest: true` 决定；发版后 10 分钟内 `dl.*` 还可能给旧版。

---

## 8. publish.yml：每天 00:00 UTC 的 cron

`publish.yml:3-11`：`schedule: '0 0 * * *'` + `workflow_dispatch` + push 到 **`master`** 且只改了 `publish.yml` 自己时触发。如果本仓库默认分支是 `main`（kamevpn-ci、00000vpn-ci、88888vpn-ci），push 触发永远不会命中，只剩 cron 和手动。

唯一 job `upload-to-release`（ubuntu-24.04）：检出私有仓库（`:24-28`，后续未用到），然后 `JasonWei512/Upload-Microsoft-Store-MSIX-Package-to-GitHub-Release@v1`（`:30-36`）按 `PARTNER_STORE` 从 Microsoft Store 拉回**已上架**的 MSIX，以 `<PKCS12_NAME>_{version}_{arch}` 命名传到 Release；`continue-on-error: true`。

为什么 `continue-on-error`：只有 Store 上真的有已发布包时它才成功；Windows 目前走便携 zip，所以这一步几乎每天失败，`continue-on-error` 让运行记录保持绿色。代价是每天 1 分钟 ubuntu 时长（2026-09-07 `gh api` 统计 30 天：每个 ci 仓库 30 次 × 1 分钟）。

---

## 9. 与母版的差异

`diff` 对照 `BuddhaJumpApp/buddhajump-ci`（2026-09-07）：

| 文件 | 差异 |
|---|---|
| `build.yml`、`publish.yml` | 0 行 |
| `build-android.yml`、`build-windows.yml` | 0 行（`kamevpn-ci`/`88888vpn-ci` 只多第 1 行注释 `# CI build workflows for <brand>, copied from ninjashield-ci.`） |
| `build-apple.yml` | **缺**母版 `170-306` 行的「Build notarized DMG (macOS direct distribution)」+「Upload DMG to GitHub release」两段（138 行）。其余逐字节相同。 |
| `build-android-hotfix.yml` | **本仓库没有**。 |

两段缺失的含义：

1. **没有 notarized DMG**：macOS 用户拿不到可以直接下载安装的包（§4.3 末尾）。母版那一段写死了 `BuddhaJump_DevID_macOS.provisionprofile` / `BuddhaJump_Service_DevID_macOS.provisionprofile`（母版 `build-apple.yml:228, 230`），移植时必须改成本品牌的 profile 名，并配好 3 个 Developer ID secret。
2. **没有 Android hotfix 侧车**：不能在不碰 Play、不碰 latest 的前提下给测试员出一个 APK。文件本身是品牌无关的，直接拷过来即可。

其他 10 个 fork 彼此完全相同（除那一行头注释）。母版 workflow 一改，10 个 fork 要逐个手工同步。

---

## 10. 已知坑（代码里有证据的）

1. **Play 上传没有开关**（`build-android.yml:115-128`）。每次 `build.yml` 都会推 AAB 到 Play internal。
2. **`build_windows_store_package` 会真的发布**（`build-windows.yml:199`）。
3. **tag 里的 `+N` 不等于安装包的 build number**（§3）。
4. **Flutter 与 Xcode 都没锁版本**（`channel: stable`、`xcode-version: latest-stable`）。
5. **Windows 构建不跑 `gen-l10n`**（§4.2 第 3 点）：源码仓库必须把 `lib/l10n/app_localizations*.dart` 生成物一起提交（各品牌源码仓库 `l10n.yaml` 的 `output-dir: lib/l10n`，生成物是 git 跟踪的）。
6. **VC++ 运行库必须随包发**（`build-windows.yml:109-139`），本地测不出来。
7. **Apple bundle id 前缀与 Android 不同**：以 kamevpn 为例，`ios/Runner.xcodeproj/project.pbxproj` 的 `PRODUCT_BUNDLE_IDENTIFIER` 是 `com.fjolsky.kamevpn`（`.service` 后缀为扩展），而 `android/app/build.gradle.kts` 的 `applicationId` 是 `com.fjolskylduoryggisverndar.kamevpn`。00000vpn 是 `com.fjolsky.vpn00000` / `com.fjolskylduoryggisverndar.fivezerovpn`。配 `APPLE_PROVISIONING_BASE64` 时按 pbxproj 里的值来，不要按 Android 包名猜。
8. **HEAD 不是 formatter-clean**：母版源码仓库上 `dart format --output=none --set-exit-if-changed lib` 会改 105 个文件里的 18 个；fork 源码同源，别跑 `dart format`。
9. **fork 不带 sing-box 规则集、端点发现被短路**：kamevpn/00000vpn 源码仓库 `git ls-files assets/rules` 为 0；`lib/constants.dart:22-25` 的 `apiBaseUrl` 默认值非空，`lib/network.dart:171-174`、`lib/pages/landing.dart:119` 以 `isNotEmpty` 走「显式配置」分支，跳过 bit.ly → GitHub 发现链。这两点都与 CI 无关，但决定了产物的行为差异。
10. **Actions 分钟数**：两个账号都是 Free 计划，公开仓库不计费。2026-09-07 `gh api` 按 GitHub 倍率（macOS ×10、Windows ×2）统计 30 天：一个 fork 一次四平台 `build` 折合 ≈132–143 计费分钟（aiglefree-ci：6 次 790 分钟）。转私有前先算清楚。
11. **公开 Release URL 曾被当作举证材料**：`maskaura-ci` 的 `https://github.com/fjolskylduoryggisverndar/maskaura-ci/releases` 被写进 Microsoft Defender 误报申诉（本地文件 `project/fjolsky/maskaura-false-positive-submission.md`）。ci 仓库转私有或改名会让这类举证失效。
12. **Android JNA 两层锁**（源码仓库 `android/app/build.gradle.kts` 固定 `jna:5.17.0@aar`，`proguard-rules.pro` 有 `-keep class com.sun.jna.**`）、**macOS 主 App 不能带 `network.server`**（`macos/Runner/Release.entitlements` 注释）、**Win32 `Create()` 会先跑一次 `OnDestroy()`**（`windows/runner/flutter_window.cpp` 的守卫）——这三条在母版源码里有证据，fork 源码是从母版回移的，回移时别漏。

---

## 11. 来自运维记忆（未在代码中复核）

1. `ORG_TOKEN` 是 classic PAT。
2. 用 shell heredoc 写 Dart 文件会把 `$` 转义掉，曾发出连不上的包。
3. Developer ID 直发的 macOS VPN 必须用 System Extension 而不是 appex；母版的 DMG 段签的是 appex，移植前先确认这条。
4. 版本号规则：补丁位 +1，到 9 后进位次版本号。
5. Play 内部测试轨道的测试者只能在 Play Console 手点。
6. Android 上引擎日志由 hydra 下发的配置关闭；客户端侧的 `CallbackThreadInitializer` 修复在母版 `VPNService.kt`，fork 是否已回移未逐个核对。
7. Apple bundle id 一经发布不能改。
8. `gh` 的活跃账号可能被其他并行会话切换；对 BuddhaJumpApp org 下仓库的 `gh workflow run` 依赖活跃账号。
9. 后端 `pkgs` 表里可能写死了 GitHub Release URL，改 ci 归属前要核对。
10. 7 个仍在 BuddhaJumpApp 下的 `<brand>-ci` 要等下载链路迁移（R2）后再搬到 fjolskylduoryggisverndar；搬完要同步改 Worker `TABLE` 和 `PRIVATE_REPO`。
