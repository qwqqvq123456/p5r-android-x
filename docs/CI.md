# 本仓库的自动化（CI）

这个 fork 自带两个工作流，全部在 GitHub Actions 上完成，不需要本地环境。

## 1. Release APK —— 构建并发布带版本号的 Release

文件：`.github/workflows/release-apk.yml`

三种触发方式：

| 触发 | 版本号来源 | 会发生什么 |
| --- | --- | --- |
| 同步工作流合并了上游（自动） | `patch` 递增 | 见下面的「同步 → 发版」链路 |
| 推送 `v*` 标签（`git tag v0.5.14 && git push origin v0.5.14`） | 标签名 | 直接用该提交构建，发布同名 Release |
| Actions → **Release APK** → Run workflow | `android:versionName` 按 `patch / minor / major` 递增，或手填 `version` | 改 `games/p5r/AndroidManifest.xml`（`versionCode` +1）→ 构建 → 提交版本号 → 发布 Release（标签在发布时创建） |

每次构建在打包前都会做一遍 **APK 校验**（`Verify the APK` 步骤），任何一项不过就中断发布、不会推到 Release：

- `aapt2 dump badging`：包名 `io.github.p5rrecomp`、`versionName` 与本次版本一致、`minSdk 29`、`native-code arm64-v8a`
- `apksigner verify`：签名链有效（打印证书 DN）
- `zipalign -c -p 4`：4 字节对齐
- `libp5r.so`、`libp5rimport.so`、`libspirvharness.so`、`classes.dex`、`AndroidManifest.xml` 都在包里

校验结果同时写进该次运行 Summary，方便回溯。

产物：

- `p5r-<版本>.apk`（arm64-v8a，Android 10+）
- `p5r-<版本>.apk.sha256`
- 同时在 Actions 的 artifact 里保留一份

Release 说明由「固定说明 + 上一个标签以来的提交列表」生成。若同版本标签已存在，手动运行会直接失败，避免覆盖已发布版本。

### 签名（可选）

不配置时用 runner 临时生成的 debug key 签名。想用固定签名（升级安装不冲突），在
**Settings → Secrets and variables → Actions** 加：

- `P5R_KEYSTORE_BASE64`：`base64 -w0 p5r.jks`
- `P5R_KEYSTORE_PASSWORD`、`P5R_KEY_ALIAS`、`P5R_KEY_PASSWORD`

keystore 可以这样生成一次：

```bash
keytool -genkeypair -v -keystore p5r.jks -alias p5r -keyalg RSA -keysize 2048 -validity 10000
base64 -w0 p5r.jks   # 贴进 P5R_KEYSTORE_BASE64
```

## 2. Sync upstream —— 自动同步上游主线

文件：`.github/workflows/sync-upstream.yml`

- 每天 03:00 UTC（北京时间 11:00）自动跑一次，也可以在 Actions 里手动运行。
- 上游默认是 `Brandon04Gonzalez/p5r-android` 的 `main`。要换上游，加仓库变量
  `UPSTREAM_REPO`（`owner/repo`）或 `UPSTREAM_BRANCH`；手动运行时也可以直接填 `upstream_repo` /
  `upstream_branch` 临时指定（做验证时很有用）。
- **能干净合并**：直接把 merge 提交推到 `main`（本 fork 自己的改动会保留），随后**自动触发 Release APK**
  （patch 递增 → 构建 → 打标签 → 发布）。即「上游更新 → 新版本 Release」全自动，无需人工点。
- **有冲突**：不硬合，而是把上游分支推到 `sync/upstream` 并开一个草稿 PR，在 PR 页面点
  *Resolve conflicts* 用网页编辑器解决；冲突未解决前不会发版。
- 手动运行时 `publish_release` 默认勾选（合并后发版），取消勾选则只合并。

如果默认的 `GITHUB_TOKEN` 没有推送权限（分支保护、组织策略），加一个 secret `SYNC_TOKEN`：
一个带 `contents` 和 `pull-requests` 写权限的 PAT，工作流会自动优先使用它。

## 关于振动功能（点击虚拟按键）

虚拟按键按下时的触感在 `runtime/src/platform_android.cpp`（`plat::haptic_tap`）里实现，
在 `runtime/src/platform_android_input.cpp` 的触摸按下处调用；强度由启动器里的
「按键震动 / Button vibration」写入 `config.txt` 的 `NX_TOUCH_HAPTICS`（0 关闭，默认 60）。
