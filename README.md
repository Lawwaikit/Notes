# Lawwaikit 的个人笔记

> 一套用 Obsidian 维护、自动发布到 GitBook 的个人知识库。

- 📖 线上站点：https://lawwaikit.gitbook.io/notes
- 🔄 发布方式：推送到 GitHub（`main` 分支）后，GitBook 自动同步构建，**无需本地编译**。

## 目录结构

```
Notes/                  # 全部笔记内容（Markdown）
├── 00-README.md       # 顶层导航
├── 01-编程语言/        # Python、Go
├── 02-测试工程/        # 测试设计、CI/CD、自动化
├── 03-AI人工智能/      # MCP 等
└── 99-计算机基础/      # 操作系统、网络、数据库
SUMMARY.md             # 站点目录（GitBook 据此生成导航）
README.md              # 本文件，同时是站点首页
introduction.md        # 「介绍」页（SUMMARY 的第一项）
zimages/               # 图片资源（已并入 Notes/ 内）
```

## 写作与组织约定

- **每个文件夹的首页命名为 `00-README.md`**：由 Obsidian 的 `folder-notes` 插件识别为文件夹首页，GitBook 也据此渲染章节落地页。
- **图片**统一放在 `Notes/zimages/`，正文用标准 `![](...)` 引用；不使用 Obsidian 专属的 `![[...]]` 语法，以保证 GitBook 能正常显示。
- 站点的目录结构完全由 `SUMMARY.md` 决定，新增或调整章节后记得同步更新它。

## 本地查看

- 无需本地构建：内容推送到 GitHub 后由 GitBook 云端渲染。
- 若要本地预览渲染效果，可访问线上站点，或使用支持 `SUMMARY.md` 的静态站点生成器。

## 技术说明

- 本站已停用本地 HonKit / GitBook CLI 构建，仅保留 GitBook SaaS 同步工作流。
- 早期用于生成 `SUMMARY.md` 的本地脚本 `gitbook-plugin-summary.py`、以及 `book.json`、`_book/` 等构建遗留均已清理。
