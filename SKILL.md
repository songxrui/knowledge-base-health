---
name: knowledge-base-health
description: |
  知识库健康巡检与自动维护。检测去重候选、孤点文件、INDEX.md 过期、交叉引用断裂，输出修复清单。
  
  适用场景:
  - 定期(每1-2周)知识库健康检查
  - 发现重复内容或孤点文件需要清理
  - INDEX.md 与实际目录结构不一致需要更新
  - 跨目录内容去重分析(alchemist vs KB主库)
  - 文件分类归并检查(裸露文件、错放目录)
  
  不适用场景:
  - 单篇文章编辑 → dbs-content / khazix-writer
  - Skill 评测 → skill-review
  - Git 操作 → 直接用 git 命令
  - 新内容创作 → dbs-content-system
  
  触发词: KB健康, 知识库检查, 去重检测, 孤点检测, INDEX更新,
           知识库维护, KB audit, 知识库巡检, 内容去重, 交叉引用检查
  
  边界: knowledge-base-health 只读扫描+输出报告(不自动删除/移动文件)。
        neat-freak 做文件整理，storage-analyzer 做存储分析。
        compile-and-verify 做内容验证，knowledge-base-health 做结构健康。
  
  正例:
  - "检查知识库有没有重复内容" → 触发
  - "更新INDEX.md" → 触发
  - "帮我做知识库月度健康巡检" → 触发
  
  反例:
  - "删除重复文件" → 先触发本 skill 做检测，再手动执行删除(本 skill 不自动删)
  - "这篇文章写得怎么样" → dbs-content
  - "整理文件夹" → neat-freak
---

# Knowledge Base Health — 知识库健康巡检 v1.0

## 环境
- 知识库根: `D:\KnowledgeBase\`
- INDEX.md: `D:\KnowledgeBase\INDEX.md`
- 排除目录: `.git`, `node_modules`, `_logs/heartbeat`(变动频繁不报警)
- 编码: UTF-8
- 命令: PowerShell

## 巡检流程

### Step 1: 结构扫描
   - 统计: 总目录数、总文件数(排除 .git)、裸露文件(根目录下非 .gitignore/INDEX.md/README.md 的孤立文件)
   - 验证 PARA 结构: 检查 `00_Inbox`/`01_Projects`/`02_Areas`/`03_Resources`/`04_Archive` 是否存在
   - 验证核心目录: `cards/` `media/` `_content-system/` `_logs/` `_meta/` 存在性
   - 输出: `HEALTH_SCAN.md`

### Step 2: 去重检测
   扫描策略:
   a. 同名文件检测: 全库搜索同名 .md 文件 → 输出路径列表
   b. 内容相似度检测(轻量): 对 .md 文件提取标题(首个 `# ` 行) → 标记完全相同标题的文件对
   c. 跨目录重复: 重点对比 `_alchemist/` 和 `media/` 下的同名/同标题文件
   d. 卡片重复: `cards/` 中文件名相似度 >80% 的标记为去重候选
   
   **不读取全文**(节省 token)——仅基于文件名+标题行做匹配

### Step 3: 孤点检测
   定义: 孤点 = 文件存在但未被任何其他文件引用
   检测方法:
   - 对每个 .md 文件: 提取文件名(不含扩展名)
   - 搜索全库: 是否有其他文件引用了该文件名(通过 `rg --files-with-matches` 或等效搜索)
   - 未被引用 + 非 INDEX.md/非日志/非配置 → 标记孤点
   - 排除: `_logs/` 下文件、`.gitignore`、frontmatter 文件

### Step 4: INDEX.md 一致性
   - 读取 INDEX.md → 提取所有列出的目录和文件引用
   - 与实际目录结构对比:
     a. INDEX 中有但目录中不存在 → 标记 "过期引用"
     b. 目录中有但 INDEX 中无 → 标记 "缺失条目"
     c. 统计数字不符(如 INDEX 写"102篇"但实际不同) → 标记 "数字过期"

### Step 5: 交叉引用断裂检测
   - 搜索所有 .md 文件中的 `[文件名]` 或 `[[文件名]]` 引用
   - 检查每个引用的目标文件是否存在
   - 不存在的引用 → 标记 "断裂引用"
   - 排除: 外部 URL、http 链接

### Step 5.5: 巡检验证
   - 去重候选验证: `$dupCount -gt 0` → 存在重复=需要人工审核，`$dupCount -eq 0` → 通过
   - 孤点验证: `$orphanCount -eq 0` → 通过，`>0` → 需清理
   - INDEX一致性: `$indexMismatch -eq 0` → 通过，`>0` → 需更新

### Step 6: 输出巡检报告

```markdown
# 知识库健康巡检报告
> 扫描时间: [YYYY-MM-DD HH:MM] | 根目录: D:\KnowledgeBase\ | 扫描文件数: N

## 结构概览
| 指标 | 值 |
|------|-----|
| 总目录数 | N |
| 总文件数 | N |
| 裸露文件(根目录) | N [列出] |
| PARA 结构完整 | ✅/❌ [缺失项] |
| 核心目录完整 | ✅/❌ [缺失项] |

## 去重候选(Top 20, 按相似度降序)
| # | 文件A | 文件B | 相似类型 | 建议 |
|---|-------|-------|---------|------|
| 1 | path | path | 同名/同标题/内容 | 保留A删B/手动检查 |

## 孤点文件(N个)
| 文件 | 类型 | 最后修改 | 建议 |
|------|------|---------|------|
| path | 卡/文章/草稿 | date | 归档/删除/建立引用 |

## INDEX.md 一致性
| 问题类型 | 详情 | 建议修复 |
|---------|------|---------|
| 过期引用 | INDEX 列出 X 但文件不存在 | 删除 INDEX 中该条目 |
| 缺失条目 | 文件存在但 INDEX 未收录 | 添加到 INDEX |
| 数字过期 | INDEX 称 N 篇，实际 M 篇 | 更新数字 |

## 断裂引用(N个)
| 源文件:行 | 引用目标 | 状态 |
|-----------|---------|------|
| path:line | [[target]] | 目标不存在 |

## 修复优先级(Top 10)
| 优先级 | 问题 | 影响 | 修复动作 |
|--------|------|------|---------|
```

## 失败兜底
| 失败 | 动作 |
|------|------|
| 知识库根目录不存在 | 输出 "D:\KnowledgeBase\ 不可达" → 中止 |
| rg 命令不可用 | 降级为 PowerShell `Select-String` |
| INDEX.md 不存在 | 输出 "INDEX.md 缺失 → 建议创建" → 继续其他巡检 |
| 文件读取权限不足 | 跳过该文件 → 记入错误清单 |

## 约束
- **只读不写**: 不自动删除/移动/修改任何文件
- **不读全文**: 仅基于文件名+标题行做匹配(节省 token)
- 修复动作需人工确认: 报告中的"建议"不自动执行
- 每 1-2 周执行一次(不频繁)

---
## 版本
- v1.0 | 2026-06-07 | 初始版本(终极形态) | 原因: KB 膨胀缺少系统级健康巡检，INDEX.md 长期未更新
## R2增强: 自动修复建议 + INDEX.md 自动更新

### INDEX.md 自动修复
触发: "修复INDEX" / "sync INDEX"
对比 INDEX.md 与实际目录 → 自动生成补丁:
- 缺失条目: 自动追加到 INDEX.md 对应章节
- 过期引用: 自动注释(`<!-- 已删除 -->`)或移除
- 数字过期: 自动更新统计数字
输出 diff → 等待用户确认 → 应用

### KB 增长趋势
读取 `_logs/reports/` 中的历史巡检报告 → 生成趋势图(mermaid):
```mermaid
xychart-beta
  title "KB增长趋势"
  x-axis ["W1","W2","W3","W4"]
  y-axis "文件数"
  bar [N1,N2,N3,N4]
```