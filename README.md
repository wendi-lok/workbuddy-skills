# workbuddy-skills

个人自建的 WorkBuddy 技能集合。每个子目录是一个独立技能，目录名即技能名。

## 技能列表

| 技能 | 说明 |
| --- | --- |
| [obsidian-paper-notes](obsidian-paper-notes/SKILL.md) | 把英文论文/技术长文的 PDF 整理成中文 Obsidian 学习笔记：双链 + callout + LaTeX + 自制 SVG 插图，按主题建文件夹写入笔记库 |
| [cn-image-host-upload](cn-image-host-upload/SKILL.md) | 把本地图片上传到**国内可直连**的免费图床并拿到公网直链。含 2026 实测可用/已失效图床清单、手写 multipart、urllib 重定向坑；给出「适配器链 + 自动降级」的推荐结构 |
| [pyinstaller-win-portable](pyinstaller-win-portable/SKILL.md) | 把 Python 项目打包成 Windows **免 Python** 绿色版 EXE 并可分发。含 onedir/onefile 取舍、`sys._MEIPASS` 与用户数据目录分离、`--add-data` 分号坑、cmd 与 zip 的中文编码坑 |

## 安装

三选一：

**1. 对话导入（推荐）**
把技能目录打包成 `.zip` 后，在 WorkBuddy 里走 `专家 → 技能 → 添加技能 → 导入技能`，拖入 zip 或文件夹即可。
> 包内必须包含 `SKILL.md`，且 frontmatter 里有 `name` 和 `description`。

**2. 手动安装（用户级，全局可用）**
```bash
mkdir -p ~/.workbuddy/skills
cp -r <技能名> ~/.workbuddy/skills/
```
Windows 对应路径：`C:\Users\<用户名>\.workbuddy\skills\`，放好后重启 WorkBuddy 即可被识别。

**3. 项目级（只在该项目生效，可随仓库共享）**
```bash
mkdir -p <项目目录>/.workbuddy/skills
cp -r <技能名> <项目目录>/.workbuddy/skills/
```

## 目录结构

```
workbuddy-skills/
├── obsidian-paper-notes/
│   └── SKILL.md          # 技能定义（YAML frontmatter + Markdown 正文）
├── cn-image-host-upload/
│   └── SKILL.md
└── pyinstaller-win-portable/
    └── SKILL.md
```

## 说明

- 技能都由本人日常使用中沉淀而来，偏实用、偏工程细节。
- 技能里的接口时效性、实测结论都标注了时间，**接口类内容会失效，用之前先复核**。
- 欢迎提 issue 交流，但请先确认你的场景与本技能的适用范围一致。
