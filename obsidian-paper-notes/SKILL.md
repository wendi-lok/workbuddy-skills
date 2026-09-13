---
name: obsidian-paper-notes
description: "把英文论文/技术长文的 PDF 整理成中文 Obsidian 学习笔记（双链 + callout + LaTeX + 自制 SVG 插图），并按主题建文件夹写入指定笔记库。当用户说「把论文总结成笔记 / 做成 obsidian 笔记 / 论文学习笔记 / 读论文整理成 md」，或给出论文 PDF 加一份写着要求的说明文件时使用。触发词：论文笔记、论文总结、obsidian、双链、插图、paper notes。"
description_en: "Turn an English paper or long PDF into structured Chinese study notes inside an Obsidian vault: wikilinks, callouts, LaTeX, and generated SVG figures, saved into a topic subfolder."
version: 1.0.0
author: wendi-lok
agent_created: true
---

# 论文 → Obsidian 学习笔记

把英文论文 PDF 变成一套「英语不好也能读懂、小白也能上手、但推导不缩水」的中文 Obsidian 笔记。

## 产出标准（默认读者设定）

| 项目 | 要求 |
| --- | --- |
| 读者 | **英语不好 + 计算机小白**。英文术语必须解释（音标 / 直译 / 比喻），难点不能靠「显然」「不难看出」糊弄 |
| 深度 | 通俗但**不切掉推导**。比如 √d_k 要写「点积方差 = d_k → softmax 饱和 → 梯度消失」 |
| 篇幅 | 不啰嗦。宁短而实，不为凑长度注水 |
| 形式 | Obsidian 原生特性：`[[双链]]`、callout、`$LaTeX$`、SVG 插图 |
| 落盘 | **不允许**堆在库根目录或上位目录，必须新建主题子文件夹 |

## 第 0 步：先读用户的「要求文件」

用户经常给一个写着要求的说明文件（比如放在某个 prompts 目录下的「论文总结.md」）。**先读它**，里面通常写明：读者水平、放到哪个库、要不要插图、能否拆成多个文件。聊天里的一句话往往不全。

## 第 1 步：环境与工具

- **Read 工具读不了 PDF**（会提示 binary）→ 先提取文本再读。
- **装 Python 包要用隔离目录**，不要污染用户环境：
  ```bash
  "$PY" -m pip install --quiet --upgrade --target "<隔离目录>" pypdf
  PYTHONPATH="<隔离目录>" "$PY" script.py
  ```
  ⚠️ 在 Git Bash 里给 `--target` 传 `/c/Users/...` 这种路径会被拼歪，可能凭空建出一棵 `C:\c\Users\...` 的垃圾目录树；**用 `C:/Users/...` 这种带盘符的写法**，并在收尾时检查有没有留下垃圾目录。
- **某些环境下 Bash 的 PATH 不完整**，`ls`/`dirname`/`find` 可能都找不到。先在命令开头补上系统目录和 Git 的 `usr/bin`，再执行：
  ```bash
  export PATH="/c/Windows/System32:/c/Windows:/path/to/PortableGit/usr/bin:$PATH"
  ```
  另外注意：**不要把 `/c/...` 形式的路径直接传给 `python.exe`**（Windows 版 Python 会把它解释成相对路径），要么用 `C:\...`，要么先 `cd` 过去。
- 如果 PowerShell 通道拿不到标准输出，就用 Bash 做文件探测，别在 PowerShell 里反复试。

## 第 2 步：提取全文

```python
from pypdf import PdfReader
buf = []
for i, p in enumerate(PdfReader(src).pages, 1):
    buf.append(f"\n\n===== PAGE {i} =====\n{p.extract_text() or ''}")
open(out, "w", encoding="utf-8").write("".join(buf))
```

15 页论文约 4 万字，一次读完不费劲。**不要跳过原文**：写笔记时数字/表格要能被原文核实。

## 第 3 步：侦察笔记库

- 摸清 Obsidian 库根，读 1～2 篇已有 md，确认：有没有 YAML frontmatter、用不用 `#tag`、标题层级、命名风格 → **跟随既有习惯**，别自作主张。
- 搜索库内有无同主题笔记，能互链就互链。

## 第 4 步：建目录

```
<库>\<上级分类>\<论文名>\
├── 00 xxx 总览.md
├── 01 ... 09 ...            ← 按论文原节次拆
├── 10 英文术语对照表.md
├── <论文编号>.pdf            ← 复制原 PDF 进来，总览里可双链打开
└── 插图\ 图1-xxx.svg ...
```

## 第 5 步：文件拆分与内容骨架

- **`00 总览`（MOC，入口）**：论文档案（标题直译/作者/会议/编号）、一分钟看懂、三项核心贡献表、关键数字、**阅读地图（双链有序列表）**、核心概念速查表、影响（后续工作）、一句话总结
- **`01～09`**：一个主题一个文件，标题就写「对应论文第 X 节」。切分建议：背景动机 / 整体结构 / 最核心的机制 / 该机制的增强版 / 配套组件 A / 配套组件 B / 训练细节 / 实验与消融 / 理论论证
- **`10 术语表`**：按主题分组的英文→中文对照（含音标、通俗解释）+ 全文公式打包
- 每篇**末尾**放 `上一站 / 下一站` 双链导航，形成环
- 正文里大量互链：`[[03 自注意力：Q K V 一次讲透|自注意力]]`

### 写作技巧（保证「小白 + 有深度」同时成立）

1. **拆词框**（英语不好的读者最需要）：
   ```markdown
   > [!note] 拆词
   > **transduction** /trænsˈdʌkʃn/ = 转导；把一个序列变成另一个序列。
   ```
2. **callout 分类型用**：`[!info]` 档案、`[!tip]` 记忆点、`[!warning]` 易错/反直觉、`[!question]` 名词解释、`[!example]` 打比方、`[!abstract]` 概念定义、`[!quote]` 原文引用
3. **公式用 `$$...$$`**（MathJax），`\begin{pmatrix}` 可用；符号第一次出现要给一句话解释
4. **打比方要落回机制**：图书馆查书（Q/K/V）、信息高速公路（残差）、秒针分针（位置编码）——比喻后面必须跟真实原因
5. **原文的坑要标出来**：例如「正文写 41.0、表格写 41.8，因为一个含检查点平均」——这类细节最能体现读懂了
6. **表格单元格里带别名的双链必须转义 `\|`**，否则表格会被打断

### 文件名注意

- 全角 `：`、`、`（U+FF1A / U+3001）在 Windows 合法，可以用
- **半角 `:` 非法**，绝对不要出现在文件名里

## 第 6 步：画插图（SVG，浅色主题）

- 手写 SVG：`viewBox="0 0 900 H"`，字体 `Microsoft YaHei, PingFang SC, sans-serif`，**每个形状显式写 `fill`**（不要依赖外部 CSS，否则可能渲染成黑色）
- 需要精确计算的图（sin/cos 曲线、热力图、坐标轴）→ **用 python 脚本生成 SVG**，生成完删脚本
- 图片嵌入用**文件名**（不写路径），前提是文件名在库内唯一：`![[图1-Transformer整体架构.svg]]`
- 正文写一句「图来自论文 Figure N，加了中文标注」

## 第 7 步：自检（必做，不能省）

**A. 链接完整性**——用 python 抽 `[[...]]` 和文件名比对（在 Windows 上 `grep | while read` 这类组合容易静默失败，别用）：
```python
import os, re
d = r"<笔记目录>"
names = set(os.listdir(d))
for f in [x for x in os.listdir(d) if x.endswith(".md")]:
    for m in re.findall(r"\[\[([^\]]+)\]\]", open(os.path.join(d, f), encoding="utf-8").read()):
        base = m.split("|")[0].split("#")[0].strip()
        if base and base not in names and not os.path.exists(os.path.join(d, base)) \
           and not os.path.exists(os.path.join(d, base + ".md")):
            print("坏链:", f, base)
```

**B. 图片渲染自检**——你看不到 SVG 的渲染结果，必须截图：
```bash
# 先写一个临时 html，用 URL 编码的 file:// 路径引用 SVG，再截图
msedge --headless=new --disable-gpu --allow-file-access-from-files \
  --screenshot="C:\tmp\check.png" --window-size=900,1500 --hide-scrollbars "file:///C:/tmp/check.html"
```
再用 Read 打开 png，检查：**文字压线、图形出界、颜色过饱和、双行标签换行**，有问题就改坐标重新生成。

> Linux/macOS 可用 `chromium --headless --screenshot` 达到同样效果。

## 第 8 步：收尾

1. 删掉全部临时文件：提取的 txt、生成脚本、`check*.html`、`check*.png`
2. 用 present_files 列出 md（**总览排第一**）+ SVG 插图，一次性批量传
3. 写工作日志到工作区的 `.workbuddy/memory/YYYY-MM-DD.md`
4. **沉淀**：把这次踩到的坑补进本技能（技能要越用越准）

## 反模式（不要做）

- ❌ 笔记堆在库根或上位目录
- ❌ 只给纯文字，不画图
- ❌ SVG 不做截图自检就交付
- ❌ 表格里写 `[[文件名|别名]]` 不转义
- ❌ 把英文术语放过去不解释，或把推导简化成「结果就是这样」
- ❌ 往笔记里写死绝对路径（库换了位置就断链）
