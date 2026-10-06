# Element X —— 用「源码 zip + GitHub Actions」构建

思路：**仓库里只提交一个源码 zip**（比逐个上传 1 万多个文件快），
由 GitHub Actions 负责「解压 → 配置环境 → 构建 → 上传 APK」。

## 目录结构

```
github-build/
├── .github/workflows/build-from-zip.yml   # 工作流：解压 zip 并构建
├── source/
│   └── element-x-src.zip                  # 源码 zip（顶层为 element-x/）
└── README.md
```

## 用法

1. 在 GitHub 新建一个仓库（空仓库即可）。
2. 把本目录下的 `.github/`、`source/`、`README.md` 推上去（**只有 3 项，很快**）：
   ```bash
   git init
   git add .github source README.md
   git commit -m "Add Element X zip build workflow"
   git branch -M main
   git remote add origin https://github.com/<你>/<仓库>.git
   git push -u origin main
   ```
3. 打开仓库 **Actions → Build Element X from ZIP → Run workflow** 即可。
   也可以在「上传新 zip 到 source/」时自动触发。
4. 构建完成后，在该次 run 的 **Artifacts** 里下载 `elementx-apk`。

## 可调参数（手动触发时）

| 输入 | 说明 | 默认 |
|---|---|---|
| `task` | Gradle 任务 | `:app:assembleGplayDebug` |
| `zip_path` | 仓库内 zip 路径 | 自动在 `source/` 下找 `*.zip` |

常用任务：
- `:app:assembleGplayDebug` —— GPlay 调试包
- `:app:assembleFDroidDebug` —— F-Droid 调试包
- `:app:assembleGplayRelease` —— GPlay 发布包（需签名配置）

## 关于源码 zip（重要）

- zip **顶层目录应为 `element-x/`**，其中包含 `settings.gradle.kts`、`gradlew` 等。
  工作流会自动定位含 `settings.gradle.kts` 的目录，所以顶层名字不敏感。
- 本目录里的 `source/element-x-src.zip` 是**瘦身版**：
  - 已剔除 `libraries/matrix/libs/*.aar`（约 104MB）；
  - 该 AAR 在 Maven Central 上有同版本 `org.matrix.rustcomponents:sdk-android:26.09.28`，
    构建时由 Gradle 自动下载，**不影响构建结果**；
  - 因此 zip 从 ~127MB 降到 ~23MB，可轻松通过 git 推送。
- 如果你**想保留本地 AAR**（体积 >100MB，会超过 GitHub 单文件 100MB 上限），
  可用以下任一方式：
  1. **分卷**：`zip -s 90m -r element-x.zip element-x`，把 `element-x.z01/z02/...` 一起提交，
     工作流会自动合并（见下）；
  2. **Git LFS**：`git lfs track 'source/*.zip'` 后再推送（工作流已开启 `lfs: true`）。

## 需要 Secrets 吗？

可选。地图/哨兵等 key 只在需要真实数据时用；不配置也能构建出可安装的 APK。
如需配置，在仓库 **Settings → Secrets and variables → Actions** 里添加：
`MAPTILER_KEY`、`MAPTILER_LIGHT_MAP_ID`、`MAPTILER_DARK_MAP_ID`、
`ELEMENT_ANDROID_SENTRY_DSN`、`ELEMENT_SDK_SENTRY_DSN`、`ELEMENT_CALL_SENTRY_DSN`、
`ELEMENT_CALL_POSTHOG_API_HOST`、`ELEMENT_CALL_POSTHOG_API_KEY`、`ELEMENT_CALL_RAGESHAKE_URL`。

## 环境（工作流已自动处理）

- JDK **21**（Temurin）
- Android SDK **platform 37 + build-tools 37.0.0**（缺失时自动安装）
- Gradle **9.8.0**（由 `gradlew` 自动下载）
- 无需 NDK（Matrix Rust SDK 以 AAR 形式提供）
