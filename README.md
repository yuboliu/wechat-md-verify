# wechat-md-verify

微信公众号文章 → Obsidian 落库前的**结构体检器**（Tier-1）。

抓取微信公众号文章（`mp.weixin.qq.com`）转 Markdown 时，提取环节会产出一些**不看原文就能判定**的
机械性缺陷。这个脚本把它们一次性查出来，作为落库前的门禁。

纯本地、零网络、零成本，**无第三方依赖**（仅 Python 标准库）。

## 检查项

| # | 检查项 | 命中含义 | 级别 |
|---|--------|----------|------|
| 1 | CODEBLOCK 占位符残留 | `__CODEBLOCK_i__` 未还原（含 markdownify 反斜杠转义形式） | FAIL |
| 2 | 重复代码块 | 多个独立块内容完全相同（**灌错内容**的强信号） | FAIL |
| 3 | 空 / 截断代码块 | 块体为空或仅 1~2 字符 | FAIL |
| 4 | 代码围栏闭合 | ``` 数量为奇数，围栏没闭合 | FAIL |
| 5 | 标题-正文粘连 | 标题里混入中文句末标点后的正文句 | WARN |
| 6 | 残留远程资源 | 残留 http 图片 / `javascript:;` / CDN 链接 | WARN |
| 7 | 图片嵌入 vs 文件 | 引用了但文件缺失=FAIL；文件多于引用=仅提示 | FAIL / 提示 |

## 用法

```bash
python3 scripts/verify_wechat_md.py "<文章>.obsidian.md"
```

可选参数：

- `--images "<自定义 images 目录>"` — 默认取 md 同级的 `images/`。
- `--strict` — 把 WARN 也视为 FAIL。

**退出码**：`0` = PASS（或仅 WARN）；`2` = FAIL。可直接接进脚本做门禁，FAIL 时禁止落库。

批量体检：

```bash
for md in output/*/*.obsidian.md; do python3 scripts/verify_wechat_md.py "$md"; done
```

## 目录结构

```
SKILL.md                      # skill 定义（Claude / WorkBuddy 等 agent 平台可读）
scripts/verify_wechat_md.py   # Tier-1 确定性检查器（核心）
references/verify_sop.md      # 完整 SOP：两层校验设计、Tier-2 升级判据、批量命令
```

## Tier-2：语义比对

Tier-1 只能证明「结构完整」，证明不了「内容对不对」。当 Tier-1 报出重复块 / 空块 / 占位符残留，
或文章是代码密集型、关键内容被渲染成截图时，需要联网拉原文做逐项 diff。判据与步骤见
[`references/verify_sop.md`](references/verify_sop.md)。

## 背景

本 skill 最初为「微信公众号文章 → Obsidian vault」的个人知识库流水线而写：

```
抓取 → 转换 → 校验(Tier-1) → 落库 → 校验(成品)
                ↑ 本仓库
```

但校验逻辑本身与具体 vault 无关，只要产物是 Obsidian 风味的 Markdown（`![[images/...]]` 嵌入 +
frontmatter）就能用。

## License

未声明。使用前请与作者联系。
