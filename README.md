# db-reverse-engineer

> 面对陌生大型业务数据库(50+ 张表)时,**3 步系统逆向出 ER 图 + 数据字典 + 重大发现报告**。
> 先扫 `information_schema` 自动分类(金额/时间/身份键/大表)再做业务 SQL — 防止"按业务直觉查表"反复漏判。

[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-c9a96a?style=flat-square)](https://claude.com/claude-code) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE) [![N=1 Verified](https://img.shields.io/badge/N=1-Verified-7fb88a?style=flat-square)](#n1-已实测案例)

## 第一性原理 · 4 句话

1. **陌生大库不能"按业务直觉"开搞** — 直觉只覆盖业务方说的表,**漏表必然**
2. **information_schema 是上帝视角** — TABLES + COLUMNS 5 秒拉完,80% 漏表风险可此阶段排除
3. **金额 + 时间 + 关联键是逆向三轴** — 字段名启发式自动分类即可,不需要业务方介入
4. **必须找四类"黄金表"**:数据字典(sys_dict)/ 权威预聚合(daily_statistics)/ 审计日志(amendment_record)/ 扩展信息(_info/_detail)

## 触发条件

| 场景 | 关键词 |
|---|---|
| 接手陌生业务库 | "全库盘点"、"陌生库怎么入手" |
| 怀疑漏表 | "我只用了 3 张表,漏了什么?" |
| 需要 ER 图/字典 | "ER 图"、"数据字典" |
| 战略口径需交叉验证 | "找权威预聚合表" |
| 找重大发现 | "找重大发现" |

## 4 步逆向法

```
Step 1 · 全扫 (5min)        information_schema.TABLES + COLUMNS
   ↓
Step 2 · 启发式分类 (10min)  金额表 / 时间字段 / 关联键 / 大表 / 黄金 4 表
   ↓
Step 3 · 批量金额时间 (15min) 每表 SUM + MIN/MAX + 异常单月检测
   ↓
Step 4 · 生成三件套 (20min)  字典 xlsx (7 sheet) + ER mermaid + 发现报告 md
```

## 13 类重大发现(N=1 提炼)

权威口径验证 · 官方字典发现 · 表族互补 · SKU 细分 · 隐藏业务 · 异常单月 · 历史回溯 · 单位异常 · 审计日志规模 · 外部系统关联 · 架构线索(MongoDB ObjectId 等) · 沉睡档案 · 空表清单

## 8 条反模式(自动扫描)

❌ 直接业务 SQL / ❌ 只看主表 / ❌ 没找 sys_dict / ❌ 假装查完 / ❌ 不查 MIN/MAX / ❌ 不查 SUM / ❌ 没识别预聚合表 / ❌ 没生成三件套

## N=1 已实测案例

| 库 | 表数 | 字段数 | 发现数 |
|---|--:|--:|--:|
| 玉佛禅寺 `temple_biz_tenant01_new` | 163 | 2,327 | **13** |

**最大收益**:发现 9 张订单表族(我此前只用 1 张)+ 权威口径交叉验证 0.4% 一致 + 风控级警报(worship 2025-12 单月¥1亿)+ sys_dict 737 字典项发现。

## 关联

- [`high-density-mckinsey-report-designer`](https://github.com/moonstachain/high-density-mckinsey-report-designer) — 摸库后写战略报告
- [`high-info-density-dashboard-designer`](https://github.com/moonstachain/high-info-density-dashboard-designer) — 摸库后做 dashboard
- [`yuanli-strategic-data-workbook`](https://github.com/moonstachain/yuanli-strategic-data-workbook) — 摸库后出 xlsx 套表
- [`yuanli-dual-layer-data-workbook`](https://github.com/moonstachain/yuanli-dual-layer-data-workbook) — 双层 xlsx

**本 skill 位于其他 skill 的"上游"** — 先 db-reverse-engineer 摸库,再 其他 skill 出洞察/报告/驾驶舱。

## 安装

```bash
git clone https://github.com/moonstachain/db-reverse-engineer ~/.claude/skills/db-reverse-engineer
```

## License

MIT
