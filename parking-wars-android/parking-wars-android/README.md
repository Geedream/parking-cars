# Parking Wars · Android 壳工程

把已经上线的 **https://project-73d17537.eazo.dev** 用 Capacitor（WebView）包成真正的安卓 App，
打出来的 APK 可以直接装到手机上：桌面图标、全屏无地址栏、独立任务栈。

游戏本体仍然跑在线上（账号、存档、AI 世界都在服务器），所以**改游戏内容不用重新打包**，
只有换图标 / 名称 / 权限 / 壳里加原生能力时才需要重新出包。

---

## 一、最快拿到 APK：GitHub Actions 云构建（不需要本地装 Android SDK）

1. 新建一个 GitHub 仓库（私有即可），把**本目录**的内容推上去（保持 `.github/workflows/` 结构）：

   ```bash
   cd parking-wars-android
   git init && git add -A && git commit -m "android shell"
   git branch -M main
   git remote add origin git@github.com:<你的账号>/parking-wars-android.git
   git push -u origin main
   ```

2. 打开仓库的 **Actions** 页 → 左侧 `Build Android APK` → **Run workflow**（push 到 main 也会自动跑）。

3. 等 4–8 分钟，页面底部 **Artifacts** 里下载：
   - `parking-wars-debug-apk` → 解开就是 `app-debug.apk`

4. 手机安装：把 APK 传到手机（微信/数据线/网盘都行）→ 点开 → 允许「安装未知来源应用」→ 完成。
   桌面出现霓虹 **P** 图标，点开就是游戏，全屏、无浏览器地址栏。

> debug APK 用公共调试签名，能装能玩，适合自用和发给朋友测试。
> 要上架 Google Play 或长期分发，用下面的签名 release 流程。

### 可选：签名的 release APK

先生成一个自己的签名证书（只需一次，务必保存好，以后升级必须用同一个）：

```bash
keytool -genkeypair -v -keystore release.jks -alias parkingwars \
  -keyalg RSA -keysize 2048 -validity 10000
base64 -w0 release.jks > release.jks.b64   # macOS: base64 -i release.jks -o release.jks.b64
```

在 GitHub 仓库 **Settings → Secrets and variables → Actions** 里加 4 个 Secret：

| Secret | 值 |
| --- | --- |
| `KEYSTORE_BASE64` | `release.jks.b64` 的全部内容 |
| `KEYSTORE_PASSWORD` | 生成证书时设的存储密码 |
| `KEY_ALIAS` | `parkingwars` |
| `KEY_PASSWORD` | 生成证书时设的 key 密码 |

然后 Run workflow 时勾上 **build_release**，产物里就会多一个 `parking-wars-release-apk`。

---

## 二、本机出包（有 Android Studio 时）

```bash
npm install
npx cap add android        # 生成 android/ 原生工程（本仓库不提交它）
npx cap sync android
npx cap open android        # 用 Android Studio 打开
```

Android Studio 里 **Build → Generate Signed App Bundle / APK** 即可，签名向导会带你建证书。

命令行版本（需要本地有 JDK 21 + Android SDK）：

```bash
cd android && ./gradlew assembleDebug
# 产物：android/app/build/outputs/apk/debug/app-debug.apk
```

---

## 三、改配置

| 要改的东西 | 改哪里 |
| --- | --- |
| 加载哪个网址 | `capacitor.config.json` → `server.url` |
| App 名称 | `capacitor.config.json` → `appName`（另外 `android/app/src/main/res/values/strings.xml` 的 `app_name` 可覆盖） |
| 包名（上架后不能再改） | `capacitor.config.json` → `appId`，生成工程前改 |
| 桌面图标 / 启动图 | `assets/icon-only.png`、`assets/icon-foreground.png`、`assets/icon-background.png`、`assets/splash.png`、`assets/splash-dark.png`，然后 `npx @capacitor/assets generate --android` |
| 启动图时长、状态栏配色 | `capacitor.config.json` → `plugins` |
| 壳内自定义行为（返回键、推送等） | `android/app/src/main/java/com/eazo/parkingwars/MainActivity.java`（生成后才有） |

生成的自定义改动写在 `android/` 里时，记得把这个目录也纳入仓库（删掉 `.gitignore` 里的 `android/`），
否则每次 CI 都会重新生成、覆盖掉你的改动。

---

## 四、已知取舍

- **WebView 壳**：游戏完全依赖网络，断网时显示内置的「连不上服务器」页（`www/index.html`）。
- **返回键**：默认走 WebView 历史，首页再按会退出 App；如果想改成「回到停车场」，在 `MainActivity` 里接管 `onBackPressed`。
- **热更新**：因为加载的是线上地址，改游戏直接发布网页即可，用户不用更新 APK；
  但千万不要把 `server.url` 指到一个会 404 的地址，否则壳会白屏。
- **上架**：Google Play 要求 targetSdk 满足当年最低要求（Capacitor 6 默认 34，随年份升级 `android/variables.gradle`）。
