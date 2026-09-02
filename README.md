# PhotonCamera-nx733j 自动构建

本仓库**不存放相机源码**，只负责自动化：每天检查上游 [bjzhou/PhotonCamera](https://github.com/bjzhou/PhotonCamera) 的最新 release，一旦出现新版本，拉取该 tag 的源码，cherry-pick [kindle4jerry/PhotonCamera](https://github.com/kindle4jerry/PhotonCamera) `main-nx733j` 分支**最新 3 个提交**（NX733J 物理可变光圈适配），编译并发布带可变光圈的 release APK。

## 工作原理

```
定时(每天) / 手动触发
        │
        ▼
查询 bjzhou/PhotonCamera 最新 release tag ── 与 last-built-release.txt 相同？ ── 是 ──▶ 跳过（不执行构建）
        │ 否
        ▼
git clone --depth 1 --branch <tag> https://github.com/bjzhou/PhotonCamera.git src
        │
        ▼
git fetch https://github.com/kindle4jerry/PhotonCamera.git main-nx733j
        │
        ▼
cherry-pick main-nx733j 最新 3 个提交（-X theirs 自动解决冲突）
  · 2a36d78b feat: variable aperture（物理可变光圈，F 按钮 + 9 档标尺）
  · a5b18dbc chore: default flavor package name to com.hinnka.mycamera.nx733j
  · e7ef81a3 fix: exif aperture follows variable aperture level
        │
        ▼
./gradlew assembleDefaultRelease（R8 混淆 + 资源压缩 + 签名）
        │
        ▼
发布 GitHub Release：tag = nx733j-<上游tag>，附带 app-default-release.apk
        │
        ▼
更新 last-built-release.txt 并提交（下次定时任务据此跳过已构建版本）
```

## 首次配置（只需一次）

1. 把本仓库推送到你自己的 GitHub（例如 `github.com/kindle4jerry/PhotonCamera-nx733j`）。
2. 在仓库 **Settings → Secrets and variables → Actions** 配置以下 Secrets：

   | Secret | 内容 |
   |---|---|
   | `RELEASE_KEYSTORE` | release 签名密钥文件的 **base64**（生成：`[Convert]::ToBase64String([IO.File]::ReadAllBytes("release-key.jks"))`） |
   | `RELEASE_STORE_PASSWORD` | 密钥库密码 |
   | `RELEASE_KEY_ALIAS` | 密钥别名（如 `photoncamera`） |
   | `RELEASE_KEY_PASSWORD` | 密钥密码 |
   | `BUILT_IN_API_KEY`（可选） | 上游 API Key，没有可留空 |
   | `BUILT_IN_API_KEY_GOOGLE`（可选） | 上游 Google 渠道 API Key，没有可留空 |

   > 签名密钥必须与之前发布的 NX733J 版本一致，否则无法覆盖安装更新。
3. 手动跑一次验证：**Actions → Build NX733J Variable Aperture Release → Run workflow**。

## 输出

- **GitHub Release**：tag `nx733j-<上游tag>`，附件 `app-default-release.apk`（包名 `com.hinnka.mycamera.nx733j`）；
- **`last-built-release.txt`**：记录已构建的上游 release tag（由 workflow 自动提交），作为每日检查的跳过依据。

## 维护 main-nx733j 适配提交

上游每次发布新版本后，需要把 3 个适配提交重新 rebase/cherry-pick 到 `kindle4jerry/PhotonCamera` 的 `main-nx733j` 分支顶端（参考上一版的手动流程）。本 workflow **永远取该分支最新的 3 个提交**，因此：

- `main-nx733j` 分支顶端 3 个提交必须始终是：可变光圈 feat、包名 chore、EXIF fix；
- 冲突时 workflow 使用 `-X theirs` 以适配提交为准自动解决；若出现无法自动解决的冲突（如文件增删）或编译失败，Action 会失败并在日志中暴露，届时人工处理并更新 `main-nx733j` 后重新触发。

## 常见问题

- **上游没更新，每天的任务白跑吗？** 不会构建，job 在版本比对步骤直接跳过（几十秒结束）。
- **如何强制重建当前版本？** 手动 Run workflow（会先删除同名旧 release 再重建）。
- **构建失败怎么看？** Actions 日志按步骤输出：cherry-pick 的三个提交、冲突状态、Gradle 编译错误都会明确显示。
