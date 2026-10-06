# 用 GitHub Actions 构建 Release APK

本仓库的 `Android Release` 工作流使用固定的个人签名密钥，生成经过签名验证的 Release APK。构建环境使用项目自带的 Gradle Wrapper 和 JDK 21，无需在电脑上安装 Android SDK。

## 1. 准备一次性的签名配置

如果已有用于这个个人版本的签名密钥，请继续使用原来的密钥，不要重新生成。更换密钥后，Android 通常无法覆盖安装旧签名的同包名应用；正式原版也不能直接被不同签名的个人构建覆盖。操作前保留需要的数据。

密钥属于你自己，必须备份密钥文件、密码和别名。GitHub Secrets 不能替代你自己的密钥备份。

### 还没有密钥时：在 Windows PowerShell 中生成

这一步只需要 JDK 提供的 `keytool`，不需要 Android SDK。生成只做一次，后续编译全部在 GitHub 云端完成。

```powershell
$signingDir = Join-Path $env:USERPROFILE "AppMarket-signing"
New-Item -ItemType Directory -Force -Path $signingDir | Out-Null
$keystoreFile = Join-Path $signingDir "appmarket-release.p12"
if (Test-Path $keystoreFile) { throw "密钥文件已存在，请使用并备份现有密钥，不要覆盖。" }

keytool -genkeypair -v -storetype PKCS12 -keystore $keystoreFile -alias appmarket -keyalg RSA -keysize 3072 -validity 10000 -dname "CN=AppMarket Personal Build"
if ($LASTEXITCODE -ne 0) { throw "密钥生成失败。" }
```

按提示设置并记住密码。上述 PKCS12 文件的密钥密码与文件密码相同，别名为 `appmarket`。如提示找不到 `keytool`，请使用已安装 JDK 的 `bin/keytool.exe` 完整路径，或先安装 JDK。

将密钥的 Base64 内容复制到剪贴板：

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes($keystoreFile)) | Set-Clipboard
```

Base64 只是文件的文本编码，不是加密。只将它粘贴到 GitHub Secret 中，不要提交到仓库、Issue 或聊天。

### 添加四个仓库 Secrets

打开仓库 **Settings → Secrets and variables → Actions → New repository secret**，创建：

| Name | Secret |
| --- | --- |
| `SIGNING_KEY` | 密钥文件的完整 Base64 文本，即刚复制的剪贴板内容 |
| `KEY_STORE_PASSWORD` | 密钥文件密码 |
| `ALIAS` | 密钥别名；按上面命令生成时填写 `appmarket` |
| `KEY_PASSWORD` | 密钥密码；按上面命令生成时与 `KEY_STORE_PASSWORD` 相同 |

请按表格名称添加到 Secrets 中，不要添加到 Variables。现有工作流会先验证这四项以及私钥能否解密；失败时不会继续执行耗时的 APK 编译。

## 2. 启用并运行工作流

1. 打开自己 Fork 的 **Actions** 页面。如果显示工作流被禁用，点击 **I understand my workflows, go ahead and enable them**。
2. 选择 **Android Release**。
3. 点击 **Run workflow**，分支选 `main`。
4. 保持勾选 **Publish the APK to GitHub Releases**，再点击绿色 **Run workflow** 按钮。
5. 等待 `build` 和 `publish` 都变绿，再到 **Releases** 下载 `.apk` 文件。

手动构建生成的发布标签为 `build-运行编号-尝试编号`，不影响 APK 内部的版本号。取消发布选项时，只生成 Actions Artifact：在该次运行的 Summary 页底部下载 `AppMarket-release-...`，解压后取得 APK。

## 3. 自动构建与正式版本标签

- 推送到 `main`：构建、校验签名并上传 Artifact；仅文档变更不触发构建。
- 推送 `v*` 标签：构建同一标签指向的源码，上传 Artifact，并将 APK 发布到对应的 GitHub Release。
- 发布只使用工作流自带的 `GITHUB_TOKEN`，无需另外创建个人访问令牌。
- 已存在的 Release 不会被自动覆盖。重复发布同一标签会报错，应检查现有 Release，或为新的版本使用新的标签。

APK 之外还会上传 `SHA256SUMS.txt` 和 `apk-signature.txt`。私钥文件仅放在 Runner 的临时目录中，不属于上传内容，构建结束时会清理。

## 常见问题

- **Missing signing values**：补齐四个 Secrets；错误中的 `KEYSTORE_PASS` 对应 Secret `KEY_STORE_PASSWORD`，`KEY_ALIAS` 对应 Secret `ALIAS`。
- **Invalid base64 / empty file**：重新从真实密钥文件生成完整 Base64，不能把文件路径作为 `SIGNING_KEY`。
- **密码或别名错误**：核对最初生成密钥时使用的值；不要通过生成另一把密钥来处理旧包的更新问题。
- **Release 创建权限不足**：检查仓库或组织的 Actions 策略是否允许该发布 Job 声明的 `contents: write` 权限。
- **没有 Run workflow 按钮**：确认工作流文件已位于默认分支 `main`，且 Fork 中的 Actions 已启用。
- **更新应用版本号**：使用 `buildSrc/src/main/kotlin/ProjectConfig.kt`。Release 标签与 APK 内部的 `versionCode`、`versionName` 是不同的字段；应用版本升级时也要维护它们。

参考：[手动运行工作流](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)、[GitHub CLI 发布 Release](https://cli.github.com/manual/gh_release_create)。
