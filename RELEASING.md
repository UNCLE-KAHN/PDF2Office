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

推 `v*` tag 后，GitHub Actions 会自动构建并创建 Release（`.github/workflows/release.yml`）。
如果 Actions 不可用，就手动发版：

1. GitHub → Releases → **Draft a new release**，选 tag `vx.y.z`；
2. 标题 `PDF2Office x.y.z`，说明从 `CHANGELOG.md` 复制；
3. 上传三个附件：`PDF2Office-x.y.z-Setup.exe`、`PDF2Office-x.y.z-Portable.zip`、`SHA256SUMS.txt`；
4. 发布后回填 `README.md`「下载」一节里的版本号（如果写死了）。

## 附件内容说明

| 附件 | 用途 |
| --- | --- |
| `PDF2Office-x.y.z-Setup.exe` | 安装包：开始菜单/桌面快捷方式 + 卸载项，默认装当前用户（无需管理员） |
| `PDF2Office-x.y.z-Portable.zip` | 绿色版：解压到任意文件夹（建议英文路径）双击 `PDF2Office.exe` |
| `SHA256SUMS.txt` | 两个附件的 SHA-256，方便用户校验 |

> 说明：安装包没有数字签名，用户首次运行可能看到 SmartScreen 提示，属正常现象。
