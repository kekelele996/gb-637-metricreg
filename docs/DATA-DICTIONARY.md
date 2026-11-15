# 数据字典 metricreg

## 1 指标定义行

| 项 | 定义 | 边界 |
|---|---|---|
| 逻辑行 | 按 `\n` 切分，行尾 `\r` 去掉 | 不在 `\x0b`、`\u2028` 上切 |
| 指标名 | 字母/数字/下划线/汉字，首字符非数字 | `where`、`group`、`by`、`depends`、`ref` 为保留词 |
| 表达式 | `agg(field)`、`ref(metric)`、数字常量、括号 | 只解析结构，不求值 |
| 过滤 | `where field = value`，单字段等值 | 多条件不支持，按 `Q_WHERE_INVALID` 隔离 |
| 分组 | `group by dim, dim` | 维度必须来自 `DIMENSIONS` |
| 依赖声明 | `depends: name, name` | 与表达式实际引用双向比对 |

## 2 字段与单位

| 登记名 | 类型 | 单位 | 维度 | 历史别名 |
|---|---|---|---|---|
| order_amt | decimal | 元 | 否 | order_amount、订单金额 |
| refund_amt | decimal | 元 | 否 | refund_amount、退款金额 |
| cost_amt | decimal | 元 | 否 | cost_amount、成本金额 |
| qty | int | 件 | 否 | quantity、件数 |
| visitor_cnt | int | 人 | 否 | visitor_count、访客数 |
| order_id | string | — | 否 | order_no、订单号 |
| store_id | string | — | 是 | shop_id、门店编码 |
| channel | string | — | 是 | channel_name、渠道 |
| category | string | — | 是 | category_name、品类 |
| city | string | — | 是 | city_name、城市 |
| order_date | date | — | 是 | biz_date、业务日期 |

- 单位写在字段表里，不在表达式里写单位后缀；单位混用靠登记表收口，不做运行期换算。
- 别名与登记名等价，命中别名即视为同一字段（字段改名不影响老定义）。

## 3 数值

- 权重 `weight` 用 Decimal，2 位最小刻度，导出为字符串。
- 常量在解析时转 `Decimal` 保存为字符串，超过 2^53 的整数不经过 float 往返。
- 影响分 `score = 下游指标数 × 受影响报表权重合计`，Decimal 2 位。
- 账目合计一律整数条数，不涉及金额。

## 4 空值与哨兵

`SENTINELS` 集中定义：`""`、`-`、`--`、`—`、`/`、`n/a`、`na`、`null`、`none`、`nil`、`nan`、`未知`、`无`、`暂无`、`待定`、`不适用`、`未取数`。

- 哨兵 ≠ 0，也 ≠ 缺列；字段位置出现哨兵按 `Q_SENTINEL_VALUE` 隔离，不参与任何统计。
- 空字符串与哨兵分开判定：空白行进「空行」账，不产生诊断。

## 5 版本有效期

- 区间为半开 `[valid_from, valid_to)`；`valid_to` 留空表示未关闭，内部用 `9999-12-31` 参与比较。
- 相邻区间不算重叠：`[01-01, 04-01)` 与 `[04-01, 07-01)` 合法。
- 起止倒挂（`valid_to <= valid_from`）按 `V_INTERVAL_INVERTED` 标记，不进入区间比较。
- 版本号是正整数且必须严格递增；提交版本号 ≤ 库中最新值返回 409 `V_STALE_SUBMIT`。

## 6 去重与代表元

- 去重键：规范化签名 `name=expr|where=…|group=…|depends=…`，全角已归一、字段已换成登记名。
- 完全相同的两行：保留首次出现（行号最小），后者按 `Q_DUP_LINE` 隔离。
- 同名不同签名：保留行号最小的一条，其余入冲突表。差异种类 → 等级：
  - 聚合集合不同 → `BLOCK`
  - 表达式主体、过滤或依赖声明不同 → `REVIEW`
  - 仅分组维度不同 → `INFO`

## 7 图与环

- 节点：`metric:`、`field:`、`dim:`、`report:` 四类，节点编号前缀即类型。
- 边：`metric_metric`、`metric_field`、`metric_dim`、`report_metric` 四类。
- 环判定只看 `metric_metric` 反向可达；单点自环（`ref(自己)`）单独给 `G_SELF_REF`。
- 环路路径的起点取分量内字典序最小的指标名，路径首尾同名。

## 8 影响面

| 项 | 定义 |
|---|---|
| 下游指标 | 反向可达集合，去自身，带 visited 防环 |
| 受影响报表 | `report_usage.csv` 中引用「自身 + 下游」的报表行，未注册指标不算 |
| 权重合计 | 受影响报表 `weight` 的 Decimal 求和，2 位 |
| 影响分 | 下游指标数 × 权重合计，Decimal 2 位 |

## 9 账目

| 账 | 等式 |
|---|---|
| 行账 | 输入行 = 注册指标行 + 隔离行 + 空行 + 注释行 |
| 节点账 | 节点 = 指标 + 字段 + 维度 + 报表 |
| 边账 | 边 = 指标→指标 + 指标→字段 + 指标→维度 + 报表→指标 |
| 分量账 | 指标 = 环内指标 + 非环指标 |

隔离行 = 解析阶段被判隔离的行 ∪ 同名冲突被挡在注册表外的行；图阶段的检出挂在指标上，不改变行状态。

## 10 运行清单

`run_id`（`MR-时间戳-4位`）、`batch_id`（批次键与 run_id 的摘要前 16 位）、`batch_key`（DSL 文本 + 两份 CSV 摘要 + 规则版本的 sha256 前 32 位）、每份输入的 sha256、规则版本、参数、起止时间、耗时、阶段计数、总账。
