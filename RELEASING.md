# 发版流程（维护者用）

仓库只放源码和脚本，**exe / zip 等大文件一律作为 GitHub Release 附件上传**，不进 git。

## 1. 改版本号

- [ ] `app.py` 里的 `APP_VERSION = "x.y.z"`（安装包版本号、窗口标题都从这里取）
- [ ] `CHANGELOG.md` 顶部加一节 `## [x.y.z] — YYYY-MM-DD`

## 2. 自检 + 构建

```powershell
python selftest.py --quick                 # 全套检查见 README「自检」一节
powershell -ExecutionPolicy Bypass -File build.ps1
powershell -ExecutionPolicy Bypass -File installer\build_installer.ps1
```

## 3. 打包绿色版 + 校验值

```powershell
$v = (Select-String app.py -Pattern 'APP_VERSION\s*=\s*"([^"]+)"').Matches[0].Groups[1].Value
Compress-Archive -Path dist\PDF2Office -DestinationPath "PDF2Office-$v-Portable.zip" -Force
$setup = "installer\output\PDF2Office-$v-Setup.exe"
$files = @($setup, "PDF2Office-$v-Portable.zip")
$lines = foreach ($f in $files) { "$((Get-FileHash $f -Algorithm SHA256).Hash.ToLower())  $(Split-Path $f -Leaf)" }
$lines | Set-Content -Encoding ascii SHA256SUMS.txt
```

## 4. 提交 + 打 tag + 推送

```powershell
git add -A
git commit -m "release: x.y.z"
git tag -a vx.y.z -m "PDF2Office x.y.z"
git push origin main --tags
```

## 5. 发布 Release

推 `v*` tag 后，GitHub Actions（`.github/workflows/release.yml`）会自动构建并创建 Release。
**发版就走这一条路**：本地不构建发布包、也不上传附件。

> 为什么不本地上传：部分网络下本机到 `uploads.github.com` 的 TLS 会被重置，而
> `gh release upload --clobber` 是「先删后传」，一旦传失败 Release 就变成空的。
> 真需要手动兜底时，先用不同的附件名上传、确认成功后再删旧附件，不要直接 `--clobber`。
> 万一已经把 Release 弄空，重新跑一次 CI 即可恢复：`gh workflow run release.yml --ref vX.Y.Z`。

## 附件内容说明

| 附件 | 用途 |
| --- | --- |
| `PDF2Office-x.y.z-Setup.exe` | 安装包：开始菜单/桌面快捷方式 + 卸载项，默认装当前用户（无需管理员） |
| `PDF2Office-x.y.z-Portable.zip` | 绿色版：解压到任意文件夹（建议英文路径）双击 `PDF2Office.exe` |
| `SHA256SUMS.txt` | 两个附件的 SHA-256，方便用户校验 |

> 说明：安装包没有数字签名，用户首次运行可能看到 SmartScreen 提示，属正常现象。
