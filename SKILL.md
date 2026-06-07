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
  
  边界: 只读扫描+输出报告(不自动删除/移动)。neat-freak做文件整理，storage-analyzer做存储分析。
  
  正例: "检查知识库有没有重复内容"→触发 / "更新INDEX.md"→触发 / "月度巡检"→触发
  反例: "删除重复文件"→先检测再手动删 / "文章质量"→dbs-content / "整理文件夹"→neat-freak
---

# Knowledge Base Health v1.2

## 环境
- 根目录: `D:\KnowledgeBase\`；INDEX.md: `D:\KnowledgeBase\INDEX.md`
- 排除: `.git`, `node_modules`, `_logs/heartbeat`
- 命令: PowerShell；编码: UTF-8

## 巡检流程

### Step 1: 结构扫描
   ```powershell
   Get-ChildItem D:\KnowledgeBase -Recurse -File | Measure-Object | Select-Object Count
   ```
   - 统计: 总目录数、总文件数、根目录裸露文件(排除 .gitignore/INDEX.md)
   - 验证 PARA 结构: `00_Inbox`/`01_Projects`/`02_Areas`/`03_Resources`/`04_Archive`
   - 验证核心目录: `cards/` `media/` `_content-system/` `_logs/` `_meta/`

### Step 2: 去重检测
   - 同名文件: 全库搜索同名 .md → 标记重复路径
   - 标题去重: 提取首个 `# ` 行 → 标记相同标题
   - 跨目录重点: `_alchemist/` vs `media/` 下的同名/同标题文件
   - 不读全文(节省token)，仅文件名+标题行匹配

### Step 3: 孤点检测
   - 提取每个 .md 文件名 → 搜索全库是否有其他文件引用 `[[文件名]]`
   - 排除: `_logs/`、`.gitignore`、frontmatter 文件
   - 未被引用 → 标记孤点

### Step 4: INDEX.md 一致性
   - 读取 INDEX.md → 对比实际目录结构
   - 标记: 过期引用(INDEX有但目录无)、缺失条目(目录有但INDEX无)、数字过期

### Step 5: 交叉引用断裂检测
   - 搜索所有 `[[文件名]]` 引用 → 检查目标文件是否存在
   - 不存在 → 标记"断裂引用"；排除外部 URL

### Step 5.5: 巡检验证
   - 验证: 去重候选数 = 0 → 通过(>0 需人工审核)
   - 验证: 孤点数 = 0 → 通过(>0 需清理)
   - 验证: INDEX 不一致数 = 0 → 通过(>0 需更新)

### Step 6: 输出巡检报告
```markdown
# 知识库健康巡检报告
> 扫描: [时间] | 根: D:\KnowledgeBase\ | 文件数: N

## 结构概览 | 去重候选 | 孤点文件 | INDEX一致性 | 断裂引用
## 修复优先级(Top 10)
```

## R2增强: INDEX自动修复 + 增长趋势
- INDEX修复: 对比INDEX与实际 → 自动生成补丁(缺失/过期/数字) → 确认后应用
- 趋势: 读取历史报告 → 生成 mermaid 增长趋势图

## 失败兜底
| 失败 | 动作 |
|------|------|
| KB 根目录不可达 | 输出原因 → 中止 |
| rg 不可用 | 降级为 PowerShell `Select-String` |
| INDEX.md 不存在 | 输出缺失 → 继续其他巡检 |
| 文件权限不足 | 跳过 → 记入错误清单 |

## 约束: 只读不写(不自动删/移/改)、不读全文(文件名+标题行)、修复需人工确认

---
## 版本
- v1.2 | 2026-06-07 | R3: G3门禁修复(可执行命令)+G4布尔验证 | 原因: 收费产品门禁5/6→6/6
- v1.1 | 2026-06-07 | R2: INDEX自动修复+增长趋势
- v1.0 | 2026-06-07 | 初始版本