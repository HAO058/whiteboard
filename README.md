# 白板 · Android 版

个人使用的协作白板 Android 应用。单 Activity + WebView（Capacitor 6）承载，UI 复用网页版引擎，针对触屏重写输入层，并按屏幕宽度在手机 / 平板两套布局间切换。

- 应用名：白板　appId：`com.hao.whiteboard`
- 产物：`app-debug.apk`（约 5–7 MB，debug 签名，侧载安装，无需上架）
- 数据：全部存本地，无账号、无网络请求、无后端

---

## 一、目录结构

```
android-app/                     ← 这个目录就是 Git 仓库根目录
├─ .github/workflows/android.yml ← 云端构建流水线
├─ keystore/wb-debug.keystore    ← 固定调试签名（勿删，见下方说明）
└─ app/
   ├─ package.json               ← Capacitor 依赖
   ├─ capacitor.config.json
   └─ www/index.html             ← 全部应用逻辑（单文件）
```

`android/` 原生工程**不入库**，由流水线里的 `npx cap add android` 现生成。

---

## 二、云端构建（推荐路径）

> 仓库已就绪：<https://github.com/HAO058/whiteboard>
> 最近一次构建：Actions → run #2 全部 11 个步骤通过，产物 `whiteboard-debug-apk`（APK 约 5–6 MB，压缩包 3.4 MB）。

### 1. 触发构建

两种方式：

- 推送到 `main`（纯文档改动不会触发，已在 `paths-ignore` 里排除 `*.md`）
- 或到 Actions 页面点 **Run workflow** 手动触发

```bash
cd android-app
git add -A
git commit -m "说明这次改了什么"
git push
```

### 2. 下载 APK

<https://github.com/HAO058/whiteboard/actions> → 打开最近一次运行 → 页面底部 **Artifacts** → 下载 `whiteboard-debug-apk.zip` → 解压得到 `app-debug.apk`。

> Artifact 下载需要登录 GitHub 账号（即使是公开仓库），30 天后过期。代码本身永久保留，过期了重新跑一次即可。

### 3. 装到设备

把 APK 传到手机/平板（微信文件传输、数据线、网盘均可），点击安装，首次需允许「安装未知应用」。装完桌面出现「白板」图标。

### 4. 后续更新

改 `app/www/index.html` → 提交推送 → 流水线自动重跑 → 下载新 APK 覆盖安装。

**覆盖安装不会丢数据**，前提是签名一致——见下节。

---

## 三、为什么仓库里要放一个 keystore

Android 只允许**同签名**的 APK 覆盖安装。签名不同时系统会拒绝升级，只能先卸载，而卸载会清空应用数据（你的板子）。

GitHub 的 runner 每次都是全新环境，`~/.android/debug.keystore` 会在首次构建时随机生成。如果不干预，**每次构建的签名都不一样**，等于每次更新都要卸载重装、数据全丢。

所以仓库里固定了一份 `keystore/wb-debug.keystore`，流水线在构建前把它复制到 `~/.android/debug.keystore`，让 AGP 的默认 debug 签名使用这份固定证书。

- 口令：`android`　别名：`androiddebugkey`　有效期：30 年
- 这份密钥只用于个人侧载，不具备任何分发安全性，泄露也无妨；**但删除它会导致签名变化**，务必保留。

---

## 四、本地构建（可选，不走云端时）

需要：JDK 17、Android SDK 34、Node 20+。

```bash
cd app
npm install
npx cap add android
cp ../keystore/wb-debug.keystore ~/.android/debug.keystore
npx cap sync android
cd android && ./gradlew assembleDebug
```

产物在 `app/android/app/build/outputs/apk/debug/app-debug.apk`。

---

## 五、改了东西之后要注意

| 改动 | 需要做什么 |
| --- | --- |
| 改 UI / 逻辑（`www/index.html`） | 直接推，流水线自动 `cap copy` |
| 加 Capacitor 插件 | 改 `package.json` 并在代码里判断 `window.Capacitor` 是否存在 |
| 改 appId / 应用名 | 改 `capacitor.config.json`；appId 一改就等于换了一个应用，旧数据读不到 |
| 升级 Capacitor 大版本 | 6 → 7 需要 JDK 21 + SDK 35，同时改 workflow 里的版本与 SDK 号 |

---

## 六、已知边界

- **不联网**：没有账号、没有云同步，换设备只能靠导出 `.json` 再导入。
- **数据在应用沙箱内**：系统「清除数据」或卸载会清空，重要内容请定期导出备份。
- **仅 Android**：iOS 需 macOS + Xcode + 签名（免费账号 7 天有效期），未纳入本次范围。
- **导入的 HTML 只认带内嵌数据块的文件**：早期网页版原型的内容是代码生成的、不存在文件里，无法迁移，会给出明确提示。
