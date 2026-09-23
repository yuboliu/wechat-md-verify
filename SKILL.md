---
name: wechat-md-verify
description: "Verifies the structural integrity of WeChat Official Account (微信公众号) articles scraped into Obsidian-flavored Markdown. Use after wechat-article-to-markdown + obsidian-markdown conversion to catch deterministic extraction defects (duplicate/empty code blocks, CODEBLOCK placeholder residue, unbalanced fences, heading-body concatenation, leftover remote images, image-embed/file mismatches) before syncing to the Obsidian vault. Triggers include 校验微信文章, 抓完即校验, verify the extracted article, or before publishing or syncing any converted wechat article."
agent_created: true
---

# Wechat Md Verify（抓完即校验 · Tier-1 结构体检）

## Overview

把「抓取 → 转换 → 落库」升级为「抓取 → 转换 → **校验** → 落库」。
本 skill 提供 `scripts/verify_wechat_md.py`，对微信文章转出的 Obsidian 风味 Markdown
（`*.obsidian.md`）做一遍**纯本地、零网络、零成本**的结构性体检，抓出提取环节最常见的
机械性缺陷。这些缺陷不看原文就能判定，因此可每篇必跑。

完整流程与「何时升级到 Tier-2 联网比对原文」见 `references/verify_sop.md`。

## When to use

- 跑完 `wechat-article-to-markdown`（抓取）+ `wechat-to-obsidian-sync/scripts/to_obsidian.py`（转换）之后，落库之前。
  （`obsidian-markdown` 是通用语法规格参考，不是流程里的一步。）
- 用户说「校验一下这篇」「抓完即校验」「完整吗」「有没有漏」。
- 批量体检：对 `output/*/*.obsidian.md` 全部跑一遍。

## Quick Start

脚本无第三方依赖（仅标准库），任意 python3 均可运行。

```bash
# 从 skill 目录用「相对路径」调用，跨平台最稳
cd ~/.workbuddy/skills/wechat-md-verify
python3 scripts/verify_wechat_md.py "<obsidian.md 路径>"
```

> ⚠️ **Windows / Git Bash 注意**：`$HOME` 会展开成 `/c/Users/...`（MSYS 风格），
> 而原生 Windows Python **不认**这种路径 —— 会报
> `can't open file 'c:\c\Users\...': No such file or directory`。
> 所以别把 `$HOME/...` 直接当参数传给 Python：`cd` 进去用相对路径（如上），
> 或把脚本路径写成 `C:/Users/...`，或用 `cygpath -w` 转换。

可选参数：
- `--images "<自定义 images 目录>"`：默认取 md 同级的 `images/`。
- `--strict`：把 WARN 也视为 FAIL（退出码 2）。

退出码：`0` = PASS（或仅有 WARN，可不阻塞）；`2` = FAIL（存在需人工处理的结构缺陷）。
可做门禁：FAIL 时禁止直接落库。

## 七项检查（Tier-1）

| # | 检查项 | 命中含义 | 级别 |
|---|--------|----------|------|
| 1 | CODEBLOCK 占位符残留 | `__CODEBLOCK_i__` 未还原（含 markdownify 反斜杠转义形式） | FAIL |
| 2 | 重复代码块 | 多个独立块内容完全相同（**灌错内容**的强信号） | FAIL |
| 3 | 空 / 截断代码块 | 块体为空或仅 1~2 字符 | FAIL |
| 4 | 代码围栏闭合 | ``` 数量为奇数，围栏没闭合 | FAIL |
| 5 | 标题-正文粘连 | 标题里混入中文句末标点后的正文句 | WARN |
| 6 | 残留远程资源 | 残留 http 图片 / `javascript:;` / CDN 链接 | WARN |
| 7 | 图片嵌入 vs 文件 | 引用了但文件缺失=FAIL；文件多于引用=仅提示 | FAIL / 提示 |

判读规则：
- **FAIL → 必须处理**：占位符残留、重复块、空块、围栏不闭合、图片缺失。
- **WARN → 人工过目**：标题粘连（路径/版本号里的 `.` 已豁免，剩下基本是真的）、远程残留。
- 「文件多于引用」是微信常态（二维码/头像/装饰图/分隔线），只提示不扣分。

## 升级到 Tier-2（联网对照原文）

Tier-1 只能证明「结构完整」，证明不了「内容对不对」。满足任一条件时，用 WebFetch 拉
`mp.weixin.qq.com` 原文作为 ground-truth，逐项 diff 标题 / 代码块 / 段落首句 / 列表表格 /
图内文字：

1. Tier-1 报出重复块 / 空块 / 占位符残留（说明代码提取环节出过问题）。
2. 文章是代码/命令密集型（教程、配置、CLI 指南）——代码块 ≥ 5 个。
3. 正文出现「如下图」「见下图」但对应位置是 `![[images/...]]`，怀疑关键内容是截图。
4. 高价值文章，要求归档质量高。

详细步骤与已实测战果见 `references/verify_sop.md`。

## 一键批量体检

```bash
cd "<你的公众号项目根>"
# Windows(Git Bash) 需把 MSYS 路径转成 Windows 路径；其它平台没有 cygpath 时原样返回
winpath() { cygpath -w "$1" 2>/dev/null || printf '%s' "$1"; }
VERIFY="$(winpath "$HOME/.workbuddy/skills/wechat-md-verify/scripts/verify_wechat_md.py")"
for md in output/*/*.obsidian.md; do python3 "$VERIFY" "$md"; done
```

## Resources

- `scripts/verify_wechat_md.py` — Tier-1 确定性检查器（本 skill 核心）。
- `references/verify_sop.md` — 完整 SOP：两层校验设计、升级判据、批量命令、当前体检状态。
