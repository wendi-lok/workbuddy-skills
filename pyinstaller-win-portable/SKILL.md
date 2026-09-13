---
name: pyinstaller-win-portable
description: 把 Python 项目（尤其是纯标准库的 http.server 小工具）打包成 Windows 上不需要装 Python 的绿色版 EXE，并生成可分发 zip。涵盖 onedir/onefile 取舍、sys._MEIPASS 与用户数据目录的分离、--add-data 的分号坑、cmd 中文乱码与 zip 中文文件名乱码、以及"文档承诺了但没有 exe"这类假交付的自查。当用户问「没有 Python 能用吗」「能不能打包成 exe」「怎么做免安装版」「Release 里没有附件」时使用。
agent_created: true
---

# Windows 免 Python 绿色版打包

## 何时用

用户说「这个软件没有 python 环境可以用吗」「能不能做成 exe」「打包成免安装的」「发给别人直接能用」。
也可能表现为**间接**信号：README/文档里写了「下载 Release（无需 Python）」，但仓库里根本没有 exe、
Releases 是空的——**先核对文档承诺和实际交付物是否一致**，别直接照着文档说"有"。

## 第 0 步：先确认交付物真的存在

```bash
ls dist/ 2>/dev/null            # 有没有打包产物
curl -s https://api.github.com/repos/<o>/<r>/releases   # Releases 里有没有附件
```

文档写了但没东西，就是文档在撒谎。这种情况的修复是**补上打包**，不是改文档措辞。

## 一、先让代码支持"冻结态"

打包后 `__file__` 指向临时解压目录，直接拿它当数据目录会出问题。要分两类：

```python
if getattr(sys, 'frozen', False):
    BUNDLE_DIR = sys._MEIPASS                                    # 只读资源（模板/静态文件）
    USER_DIR = os.path.dirname(os.path.abspath(sys.executable))  # 用户数据（配置/上传/输出）
else:
    BUNDLE_DIR = USER_DIR = os.path.dirname(os.path.abspath(__file__))
```

- **只读资源**从 `sys._MEIPASS` 读（PyInstaller 把 `--add-data` 的内容放这）。
- **用户数据**放 exe 同级，用户拷文件夹就能迁移配置。
- 定位 exe 目录**用 `sys.executable`，不要用 `sys.argv[0]`**：某些启动方式下 `argv[0]` 是相对路径，
  `os.path.dirname` 会返回空串，数据目录就散到当前工作目录去了。
- 控制台输出加 `flush=True`，否则打包后看不到任何 print（缓冲）。

## 二、打包命令

```bash
pyinstaller --onedir --clean --noconfirm --name AppName \
  --add-data "templates;templates" \
  --exclude-module tkinter --exclude-module unittest --exclude-module pydoc \
  --exclude-module test --exclude-module distutils --exclude-module lib2to3 \
  app.py
```

关键取舍与坑：

- **`--onedir` 优先于 `--onefile`**。onefile 每次启动都把整个运行时解压到 `%TEMP%`，冷启动明显更慢，
  杀软也更容易误报。代价是必须整个文件夹一起分发——用 zip 解决。
- **`--add-data` 的分隔符平台相关**：Windows 是 `源;目标`（分号），Linux/macOS 是 `源:目标`（冒号）。
  写错**不会报错**，只是文件没打进去，运行时才 404。这是排查成本最高的一类坑。
- **不要排除 `email`**：`http.server` 解析请求头依赖它（`email.parser` / `email.utils`）。
  可以安全排除的是 `tkinter` / `unittest` / `pydoc` / `test` / `distutils` / `lib2to3`。
- 一个纯标准库 `http.server` 应用打出来约 23MB，zip 后约 8~9MB。

## 三、生成启动入口和说明

**`启动.bat` 内容全用 ASCII，不要写中文。** cmd 按"当前代码页"读取 bat，和脚本里的 `chcp 65001`
互相打架，很容易乱码。提示语用英文即可：

```bat
@echo off
cd /d "%~dp0"
title AppName
echo Starting AppName ...
echo Close this window to stop the server.
echo.
AppName.exe
if errorlevel 1 (
    echo.
    echo [ERROR] Failed to start. Please read the message above.
    pause
)
```

**说明文件（`使用说明.txt`）不要用 `echo 中文 > 文件` 生成**——cmd 重定向按当前代码页写字节，必乱码。
交给 PowerShell 以 UTF-8 写：`Set-Content -Path x.txt -Value $t -Encoding UTF8`。

## 四、打 zip：中文文件名的编码坑

- ❌ **Windows PowerShell 5.1 的 `Compress-Archive` 不写 UTF-8 标志位**，中文文件名解压出来是乱码。
- ✅ 用 .NET 的带编码重载：
  ```powershell
  [System.IO.Compression.ZipFile]::CreateFromDirectory(
      $src, $dst, [System.IO.Compression.CompressionLevel]::Optimal, $true,
      [System.Text.Encoding]::UTF8)
  ```
- ✅ 或直接用 Python `zipfile`（非 ASCII 文件名会自动置 UTF-8 位）：
  ```python
  with zipfile.ZipFile(dst, 'w', zipfile.ZIP_DEFLATED, compresslevel=9) as z:
      z.write(full, rel.replace('\\', '/'))
  ```
- **校验方法**：读条目 `info.flag_bits & 0x800` 应为真，否则解压方（尤其 Windows 资源管理器）会乱码。

> 环境限制提醒：某些受管控环境里 Bash 不能调 `cmd.exe`，PowerShell 工具也不能调 `cmd.exe`，
> 且 `Add-Type` 可能被拦。这时**别硬碰**：用 Python 的 `zipfile` / `subprocess` 走等价逻辑，
> 并用"写文件再读"的方式绕过看不到 stdout 的问题。

## 五、必须做的验证

**光有 exe 不算完成，要模拟"用户下载解压后在新环境跑"。**

1. 把 zip 解压到一个全新目录（别在 build 目录里测）。
2. **剥离 Python 的 PATH** 再运行，证明真的不依赖 Python：
   ```bash
   export PATH="/c/Windows/System32:/c/Windows"   # 只留系统目录
   command -v python || echo "python NOT on PATH (good)"
   ./AppName.exe
   ```
3. 验证服务真的起来了：`curl` 首页 + 主要 API，看状态码。
4. **比对首页字节 md5 与源码模板是否一致**——这是确认"打进包的是当前代码，不是旧模板"的最快方法。
5. 确认 exe 同级自动建出了 `data/` 之类的用户目录（验证路径逻辑）。
6. 收尾：`tasklist | grep -i <exe名>` 确认没有残留进程占用端口/文件锁。

## 六、发布 Release 让文档兑现

README 写了"从 Releases 下载"，就要真的挂上附件：

```
POST /repos/{o}/{r}/releases                              → 建 release（tag_name / target_commitish）
POST https://uploads.github.com/repos/{o}/{r}/releases/{id}/assets?name=X.zip   → 传附件
```

- 附件走的是 `uploads.github.com`，和 `api.github.com` 是**两个域**。
- 校验附件状态用 API 看 `state == "uploaded"` 和 `size`；**不要用 `curl -I`**——
  下载跳转到 `objects.githubusercontent.com`（CDN），这个域在受限网络里经常连不通（`http=000`），
  会让"附件不存在"和"网络不通"看起来一模一样。**要如实说明是哪一种**。
- 同名附件要发新版时，先 `DELETE /releases/assets/{id}` 再传，脚本才能重复执行。

## 排错速查

| 现象 | 原因 |
|------|------|
| 打包后首页 404 / 模板找不到 | `--add-data` 分隔符写错，或没做 `sys._MEIPASS` 路径切换 |
| 打包后控制台无任何输出 | print 缺 `flush=True` |
| exe 双击一闪而过 | 用 `启动.bat` 启动，靠 `pause` 留住错误信息 |
| `data/` 跑到奇怪的地方 | 用了 `sys.argv[0]` 定位 exe 目录 |
| 解压后中文文件名乱码 | zip 没置 UTF-8 标志位（见第四节） |
| exe 单独拷出来起不来 | 依赖同级 `_internal/`，必须整个文件夹一起分发 |
| "已保护你的电脑"提示 | exe 无代码签名，让用户点「更多信息」→「仍要运行」 |
