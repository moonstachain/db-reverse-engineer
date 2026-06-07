---
name: db-reverse-engineer
description: 面对陌生大型业务数据库(50+ 张表)时,3 步系统逆向出 ER 图 + 数据字典 + 重大发现报告。**先扫 information_schema 自动分类(金额/时间/身份键/大表)再做业务 SQL**,防止"按业务直觉查表"反复漏判。触发关键词:"全库盘点"、"逆向工程数据库"、"ER 图"、"数据字典"、"找重大发现"、"陌生库怎么入手"、"业务数据库不知道有什么表"、"漏表了"、"db reverse engineer"、"data dictionary"。N=1 范本:某寺院 163 表逆向(13 处重大发现)。
---

# db-reverse-engineer

> **N=1 dry-run 完成**(某寺院 temple_biz_demo · 163 表 / 2,327 字段 · 13 处重大发现 · 2026-05-23)
> N=2 待复用 — 候选:任何 50+ 表的业务库逆向(OSA / yiru / 量化策略库 / 慈善基金会 / 任意 SaaS 内部库)
>
> **核心信念 · 第一性原理**:
> 1. **陌生大库不能"按业务直觉"开搞** — 直觉只覆盖业务方说的表,漏表必然(N=1 案例:连续 3 次漏判 neu_order_info / tablet_relation LSCN / neu_order_worship)。
> 2. **information_schema 是上帝视角** — 它包含全部表 + 全部字段 + 全部注释,完整且权威,5 秒拉完。
> 3. **金额 + 时间 + 关联键是逆向三轴** — 字段名启发式分类即可自动识别,不需要业务方介入。
> 4. **找四类"黄金表"是关键** — 数据字典表(sys_dict)/权威预聚合表(daily_statistics)/审计日志表(amendment_record)/扩展信息表(_info/_detail)— 它们决定逆向深度。
>
> 厉害的逆向工程 = **3 步从 0 到 ER+字典+发现报告,不依赖业务方提供 ER 图**。

## 触发条件

| 场景 | 行为 |
|---|---|
| 接手陌生业务库,不知有哪些表 | 启动 Step 1 全扫 |
| "我只用了 3 张表,这库 100+ 表都漏了什么?" | 跑 Step 2 自动分类 + Step 3 找漏表 |
| 战略报告 ¥X 亿口径,但不知是否含全部业务 | Step 3 找权威预聚合表交叉验证 |
| 业务方记忆说"老数据迁过来了"但你查不到 | Step 2 字段名扫 + 多表 join 试 |
| 需要给领导 ER 图 / 数据字典 | Step 4 自动生成 xlsx + mermaid |
| 担心漏表/漏判,要"先把库摸透再分析" | 全跑 4 步,出 13 项发现报告 |

## 第一性原理 · 4 个根问题

### Q1 · 为什么不能"按业务直觉"开搞陌生库?
**答**:业务方说的表是"业务视角主表"(如 neu_order),但实际库里有**主表 + 扩展信息表(_info)+ 流水表(_detail)+ 子单表(_worship)+ 退款表(_back)+ 日志表(_log)+ 预聚合表(_statistics)+ 审计表(amendment_record)+ 字典表(sys_dict)** 一整套。**业务直觉只覆盖第一层,后面 8 层都漏**(某寺院案例:9 张订单表族,我只看了 1 张)。

### Q2 · information_schema 能给到什么程度?
**答**:5 秒拉完 `TABLES`(行数/体积/注释)+ `COLUMNS`(字段名/类型/注释)。**配合字段名启发式 + 注释关键词搜索,80% 的"漏表风险"可在第一阶段排除**。剩下 20%(如 worship 单月异常)需要 Step 3 批量扫金额/时间发现。

### Q3 · 启发式分类有多准?
**答**:N=1 案例验证:
- 金额表识别(字段名含 price/money/amount/fee/payfee/orderMoney):**N=1 命中 11/11**
- 时间字段识别(create_time/createDate/pay_time):命中 100%
- 关联键(出现 ≥5 表的字段):自动找到 coder/phone/app_user_id 等真实 join 键
- 隐藏业务识别(注释关键词搜索):找到"莲池海会"等 6→7 业务

### Q4 · 必须找的 4 类"黄金表"是什么?
**答**:
1. **数据字典表**(sys_dict / dict / code) — 所有 code 字段含义的官方权威
2. **权威预聚合表**(daily_statistics / summary / report) — 用来交叉验证主口径,避免"算了一年发现差 10 倍"
3. **审计日志表**(amendment_record / audit_log) — business_type 字段揭露隐藏业务类型
4. **扩展信息表**(*_info / *_detail / *_extra) — 含历史数据回溯 + 与外部系统关联键(如 ddkOrderNo)

## 4 步逆向法(标准流程)

### Step 1 · information_schema 全扫(5 分钟)

```sql
-- 所有表 + 行数 + 体积 + 注释
SELECT TABLE_NAME, TABLE_ROWS,
  ROUND((DATA_LENGTH+INDEX_LENGTH)/1024/1024,2) mb,
  IFNULL(TABLE_COMMENT,'') cmt
FROM information_schema.TABLES
WHERE TABLE_SCHEMA='<DB>' AND TABLE_TYPE='BASE TABLE'
ORDER BY TABLE_ROWS DESC;

-- 所有字段 + 类型 + 注释
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE, IFNULL(COLUMN_COMMENT,'') cmt
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='<DB>'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

存 JSON 备用。**这一步给你全库视角,任何后续分析都基于此**。

### Step 2 · 启发式自动分类(10 分钟)

对每张表自动标记:

| 维度 | 识别规则 | 例 |
|---|---|---|
| **金额表** | 字段名含 `price` / `money` / `amount` / `fee` / `payfee` / `orderMoney` | neu_order / pay_log / neu_order_flowing |
| **时间字段** | 字段名含 `time` / `date` / `datetime` | create_time / createDate / pay_time |
| **关联键** | 字段名出现在 **≥5 张表** | coder(28 表)/ phone(17 表)/ app_user_id(17 表) |
| **身份键表** | 同表含 ≥2 个身份字段 | neu_order(customerId+app_user_id+union_id+phone) |
| **大表 Top 25** | 按 TABLE_ROWS 排序 | 找漏表 |
| **空表** | TABLE_ROWS = 0 | 设计未启用 |
| **黄金 4 表** | 表名/注释含 `dict` / `daily_statistics` / `amendment_record` / `_info` `_detail` | sys_dict / order_daily_statistics |

**输出**:候选清单(大表 Top 25 / 金额表 Top 15 / 身份键表 / 黄金表 4 类)。

### Step 3 · 批量金额 + 时间统计(15 分钟)

对每张金额表自动跑:

```sql
SELECT COUNT(*), ROUND(SUM(<money_field>)/10000,1) wan,
  MIN(<time_field>), MAX(<time_field>)
FROM <DB>.<table>;
```

**找异常**:
- **金额异常**:SUM 远超业务总盘 → 单位问题(元 vs 分)或脏数据
- **时间异常**:MIN 早于已知系统启用时间 → **历史数据回溯候选**(某寺院案例:皈依 1990 起 / 退款 2008 起 / 订单_info 2000 起)
- **行数比主表多 30%+** → 流水/分账表(detail/flowing)
- **行数小但金额大** → 高客单表(某寺院案例:order_flowing N 万笔 ¥X 亿,客单 ¥X 万)

对**月度热力图**(年×月 SUM):**找异常单月**(某寺院 worship 2025-12 单月 ¥X 亿 / 月均值 250 倍 = 风控警报)。

### Step 4 · 生成 ER + 字典 + 发现报告(20 分钟)

3 件套交付:

#### A · 数据字典 xlsx(7 sheet 标准结构)
| Sheet | 内容 |
|---|---|
| 00_README | 索引 + 数据基座 + 方法说明 |
| 01_表清单 | 全部表(行数/体积/注释/分类[v3已用/漏判/大表/空表]) |
| 02_字段 | 全部字段(类型/注释)·可搜 |
| 03_sys_dict_权威 | 数据字典表全量导出 |
| 04_关联键 | 高频字段 + 典型 join 用途 |
| 05_金额表 | 所有金额表 + SUM + 时间 + 战略价值 |
| 06_重大发现 | 13 项(或 N 项)发现 · 紧迫度颜色编码 |

#### B · ER 图(Mermaid)
```mermaid
erDiagram
    NEU_CUSTOMER ||--o{ NEU_ORDER : "customerId"
    NEU_ORDER ||--o{ NEU_ORDER_DETAIL : "orderCoder"
    ...
```

#### C · 重大发现报告 md
Governing Thought + SCQA + N 项发现表 + ER 图 + 给业务方核验问题清单。

## 13 类"重大发现"清单(N=1 提炼)

| 类型 | 是什么 | 某寺院案例 |
|---|---|---|
| **权威口径验证** | 找到预聚合表,与主分析口径交叉 | daily_statistics ¥5.44 亿 vs 我 ¥5.46 亿(0.4% 一致) |
| **官方字典发现** | sys_dict 等权威码表 | 737 字典项揭露所有 code 含义 |
| **表族互补** | 主表 + N 张扩展表 | 订单 9 张表族(我只用过 1 张) |
| **SKU 细分** | worship_content / sku_name 比 type 细 | 11 大类 → 几十 SKU |
| **隐藏业务** | amendment_record.business_type 揭露 | "莲池海会"是第 7 类业务 |
| **异常单月** | 月度热力图飙升 | 2025-12 单月 ¥X 亿(均值 250 倍) |
| **历史回溯** | 时间 MIN 早于已知启用时间 | 皈依 1990 / 退款 2008 |
| **单位异常** | 金额 SUM 不合理 | pay_log ¥874 亿(元 vs 分待核) |
| **审计日志规模** | sys_amendment_record 等 | 887 万行 7 业务全审计 |
| **外部系统关联** | ddkOrderNo / 第三方同步表 | 点点客 ddk 121.5 万条关联 |
| **架构线索** | 主键格式 / ObjectId | business_id 是 MongoDB ObjectId |
| **沉睡档案** | 档案表行数 > 消费用户数 | 318k 档案 vs 124k 消费 = 19 万沉睡 |
| **空表清单** | TABLE_ROWS=0 的设计意图 | 设计了但没用的功能 |

## 8 反模式 · 自动扫描清单

```
❌ 直接业务 SQL          → 应先 information_schema 全扫        → 改 Step 1
❌ 只看主表              → 应扫表族(_info/_detail/_back/_log)→ 改 Step 2
❌ 没找 sys_dict 推 code  → 应先找数据字典表                   → 改黄金4表清单
❌ 假装查完全部          → 应标注"未扫表数 N"                  → D3 不确定性
❌ 时间字段不查 MIN/MAX  → 漏掉历史回溯发现                     → 改 Step 3
❌ 金额字段不查 SUM      → 漏掉异常/单位问题                    → 改 Step 3
❌ 没识别预聚合表        → 白做交叉验证                         → 改黄金4表清单
❌ 没生成 ER/字典/报告   → 留口头结论,后续仍漏                  → 改 Step 4
```

## 6 重构模块(可单独移植)

| 模块 | 对应步骤 | 干啥 | N=1 范例 |
|---|---|---|---|
| **A · 元数据全扫脚本** | Step 1 | 拉 information_schema 存 JSON | erscan1.py |
| **B · 启发式分类器** | Step 2 | 自动标金额/时间/身份键/大表 | 字段名 regex + 关联键频次 |
| **C · 黄金 4 表雷达** | Step 2 | 自动找 dict/statistics/audit/_info | 表名/注释关键词搜 |
| **D · 金额+时间批量扫** | Step 3 | 对候选表 SUM+MIN/MAX | erscan2.py |
| **E · 异常单月检测** | Step 3 | 月度热力 SUM 飙升识别 | worship 2025-12 单月 ¥X 亿 |
| **F · 三件套生成器** | Step 4 | 字典 xlsx + ER mermaid + 发现 md | build_dict.py |

## N=1 已实测案例

| 库 | 表数 | 字段数 | 发现数 | 配套交付 | 日期 |
|---|--:|--:|--:|---|---|
| 某寺院 `temple_biz_demo` | 163 | 2,327 | **13** | 字典 xlsx(7 sheets)+ ER mermaid + 报告 md | 2026-05-23 |

**最大收益**:发现 9 张订单表族(我此前只用 1 张)+ 权威口径交叉验证(0.4% 一致)+ 风控级警报(worship 2025-12 单月 ¥X 亿)+ sys_dict 737 字典项官方权威 + **战略口径 ¥5.46 亿升级到 ¥9.7 亿**(发现老金额 LS* 系列 ¥4.26 亿已迁入 neu_order_info)。

## ⭐ 关键护栏 · 业务方一句话 > agent 10 次 SQL(v1.1 新增)

### N=1 案例的"4 次漏判同一系列数据"惨痛教训

我连续 **4 次漏判** 同一系列 LSCN 老常年供位数据,因为它分散在 5 张表的不同字段:

| 版本 | 我的判断 | 错在哪 | 业务方哪句话纠正了我 |
|---|---|---|---|
| v1 | "在 tablet_back 表" | 没找对表 | 水月质疑"不对吧" |
| v2 | "LSCN 只 2 笔,没进库" | 只查订单主表 neu_order | **胡恺:"常年供位有 9184 条"** |
| v3 | "牌位进了 9152 条,但金额没进" | 没查 neu_order_info 的 LS* 前缀 | 自己全库逆向才看到 |
| v4(终) | **"LS\* 共 18,000 条 / ¥4.26 亿已在"** | ✅ | **周诚:"已导入脱敏服务器"** |

**真相**:同一份"老常年供位"数据散在 **5 张表 / 不同字段**:
- `neu_buddhist_tablet_relation`(牌位维度,id LIKE 'LSCN%')
- `neu_order_info`(金额维度,coder LIKE 'LSCN%')
- `neu_order`(订单主表,完全没有)
- `neu_buddhist_tablet_back`(早期备份)
- `neu_customer`(斋主档案,customerCoder 100% 关联)

**每张表只露一面**。业务方一句"已导入脱敏服务器"瞬间把 5 个片段串起来。

### 持续校验原则(强制护栏)

**每次业务方补充新信息(新导入 / 字段说明 / 记忆纠正)后,必须重跑 Step 2 黄金 4 表雷达 + Step 3 金额+时间扫描** — 因为:
- 老数据可能**刚刚被导入**,你昨天查的结论今天就过期了
- 业务方的一句话**揭示的新关联键**(如 LSCN 前缀)需要重新扫全库
- "数据库逆向" 不是一次性快照,是**持续与业务对话的迭代过程**

### 反模式更新(D8 自我封闭)

```
❌ "我查过了" 自我封闭          → 应保持业务方信号→重扫的反射弧
❌ "上次查没有" 当结论          → 应每次业务方补信息后重跑黄金 4 表雷达
❌ 同一系列数据漏判 ≥2 次       → 应记录"漏判链"留痕,警示后续
```

### 给所有用本 skill 的人的 1 条死规矩

> **业务方一句话比 agent 10 次 SQL 都重要。** 任何"我查不到 = 不存在"的判断,在业务方说"应该有"时,必须**重跑全套 4 步法**,不能用历史结论辩驳。



## N=2 候选(待复用)

- OSA 战略管理库 → 找漏表 + 表族
- yiru 量化投研库 → 因子/持仓/收益表族
- 慈善基金会 / 某客户基金会 → 同寺院系列
- 任何 SaaS 内部库逆向(看产品到底有多少功能)

## 边界 · 不要越界

- ❌ **不替代业务方对话** — 字典是骨架,业务方的"为什么这么设计"是肉
- ❌ **不替代真实分析** — 这个 skill 帮你"摸库",不帮你"出洞察"(那是 high-density-mckinsey 的事)
- ❌ **小库不上 4 步法** — < 30 表的简单库,直接看 schema 就够
- ✅ **诚实标"扫了多少"** — D3 不确定性显式化(本次扫了 163/163 还是 163/300)
- ✅ **N=1 时标实验性** — 现仅一个案例,方法论需更多验证
- ✅ **金额异常必标** — 不要因为算得出 SUM 就当成结论

## 复用规范

1. **必跑 4 步全流程**(不要跳 Step 1 直接业务 SQL)
2. **必生成 7 sheet 字典 xlsx**(不要只给"我盘点了一下")
3. **必标"黄金 4 表"是否找到**(dict/statistics/audit/_info)
4. **必有"漏判风险"声明**(诚实说"扫了 X/Y,某 N 张待核")
5. **必带 N 项发现编号**(D1-D13 这种,便于业务方逐项答复)

## 关联

- `[[high-info-density-dashboard-designer]]` — 屏幕 dashboard 设计(出报告后做)
- `[[high-density-mckinsey-report-designer]]` — 战略报告设计(出报告后做)
- `[[yuanli-strategic-data-workbook]]` — 顶层 xlsx 套表(数据交付)
- `[[yuanli-dual-layer-data-workbook]]` — 双层 xlsx(数据+底表)
- 本 skill **位于他们的"上游"** — 先用 db-reverse-engineer 摸库,再用其他 skill 出洞察/报告/驾驶舱

## 与父子 skill 的位置

```
              ┌─────────────────────────────────────────────────┐
              │  db-reverse-engineer (本)                       │
              │  上游 · 摸清库结构 · 出字典 ER 发现报告          │
              └─────────────────────┬───────────────────────────┘
                                    ↓ 摸清后
              ┌─────────────────────┴───────────────────────────┐
              │  high-density-mckinsey-report-designer (报告)    │
              │  high-info-density-dashboard-designer (屏幕)     │
              │  yuanli-strategic-data-workbook (xlsx 顶层)      │
              │  yuanli-dual-layer-data-workbook (xlsx 双层)     │
              └─────────────────────────────────────────────────┘
```

## 第一性原理 · 总结(单页)

**逆向陌生大库不能靠直觉,要靠 information_schema 的上帝视角 + 启发式自动分类 + 黄金 4 表雷达。**

任何陌生库,3 步可成:
1. **全扫**(information_schema TABLES + COLUMNS)
2. **分类**(金额/时间/关联键/大表/黄金4表)
3. **统计 + 出三件套**(字典 xlsx + ER + 发现报告)

**漏判风险随表数指数增长** — 50 表漏 1 张,200 表漏 8 张。本 skill 防止漏判。

**N=1 实证**:某寺院 163 表,我此前 3 轮业务 SQL 漏 3 次,跑完本 skill 4 步法,**13 处重大发现一次性挖出**。
