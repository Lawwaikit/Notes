# MEMORY.md — 项目长期约定（D:\Documents\Notes）

## 项目本质
- 个人知识库，**发布到 GitBook SaaS**：线上站 https://lawwaikit.gitbook.io/notes ，从 GitHub 仓库 `git@github.com:Lawwaikit/Notes.git`（branch `main`）自动同步。
- 写作流：Obsidian（folder-notes 插件，`folderNoteName: "00-README"`）+ GitBook 渲染（布局/主题全在 GitBook 侧，无需本地构建）。

## 目录与文件名约定（与线上 GitBook 一致，务必遵守）
- **文件夹首页 = `00-README.md`**（不是 index.md）。
- 首屏 = 根 `introduction.md`；根 `README.md` 是 GitBook 默认 readme。
- **没有 `.gitbook.yaml`**：GitBook 直接读仓库根 `SUMMARY.md` 渲染目录。
- 图片统一放 `Notes/zimages/`，正文用标准 `![](../../zimages/...)`（不要用 Obsidian 专属 `![[...]]`，GitBook 不认）。
- `SUMMARY.md` 所有链接必须指向 `.md` 文件（含文件夹的 `.../00-README.md`），不要只写文件夹路径。

## 用户偏好
- **不喜欢 MkDocs Material 的布局/主题**，明确偏好 GitBook 的观感。后续若再谈"文档站"，默认走 GitBook，不要擅自迁 MkDocs。

## 环境限制（重要）
- **无法从本沙箱 push 到 GitHub**：SSH 22 端口超时；HTTPS 443 能连但无凭据/无 TTY 无法鉴权。任何需要 push 的活，本地提交后交由用户在本机执行 `git push origin main`。

## 历史坑
- Windows 大小写不敏感 + git：把 `notes/` 重命名为 `Notes/` 时，git 默认 `core.ignorecase=true` 会把 `Notes/` 折叠成 `notes/`，导致 SUMMARY.md（大写 `Notes/`）与 GitHub 实际路径（小写）不符、GitBook 找不到页面。→ 暂存/提交加 `git -c core.ignorecase=false add/rm` 强制保留磁盘大写。
