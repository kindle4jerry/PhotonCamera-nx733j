# PhotonCamera-nx733j 自动构建

本仓库**不存放相机源码**，只负责自动化：每天检查上游 [bjzhou/PhotonCamera](https://github.com/bjzhou/PhotonCamera) 的最新 release，一旦出现新版本，就用 NX733J 适配分支（[kindle4jerry/PhotonCamera](https://github.com/kindle4jerry/PhotonCamera) 的物理可变光圈适配）编译并发布 release APK。

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
列出所有 main-nx733j-<版本> 适配分支，取版本号最新者作为适配提交来源
（一个都没有时回退到 main-nx733j）
        │
        ▼
cherry-pick 该来源分支顶端 3 个提交（-X theirs 自动解决冲突）
  · feat: variable aperture（F 按钮 + 9 档标尺）
  · chore: default flavor package name → com.hinnka.mycamera.nx733j
  · fix: exif aperture follows variable aperture level
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

> 适配提交来源取**版本最新的适配分支**（如 `main-nx733j-1.28.0.1`），因为它的基线离当前上游最近、cherry-pick 漂移最小；
> 构建目标始终是**上游最新 tag**，因此产物 = 最新上游源码 + 最新可变光圈适配。

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

## 维护适配分支

适配提交来源 = 所有 `main-nx733j-<版本>` 分支中**版本号最新**的那个（无则回退 `main-nx733j`）。因此维护约定是：

- 上游发布新版本后，若 Action 自动 cherry-pick 后编译失败（上游重构导致，如 1.28.0 重写了 `ParameterRuler`/`CameraParameterBar`），
  就以该 tag 为基线创建/更新适配分支 `main-nx733j-<tag>`，在其上解决冲突并验证编译，然后推送；
- 该分支顶端 3 个提交必须始终是：可变光圈 feat、包名 chore、EXIF fix（workflow 取 tip-3）；
- 平时小版本更新无需人工介入：workflow 会自动用最新适配分支的 3 个提交合并到最新上游 tag。

## 常见问题

- **上游没更新，每天的任务白跑吗？** 不会构建，job 在版本比对步骤直接跳过（几十秒结束）。
- **如何强制重建当前版本？** 手动 Run workflow（会先删除同名旧 release 再重建）。
- **构建失败怎么看？** Actions 日志按步骤输出：cherry-pick 的三个提交、冲突状态、Gradle 编译错误都会明确显示。
