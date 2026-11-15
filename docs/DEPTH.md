# 深度登记 metricreg（指标定义注册与依赖环检测）

- 规模：中型（6 个功能模块 / 6 个页面 / 71 个文件 / 56 条测试）
- 核心规则一：**DSL 解析与登记**——一行一条 `name = expr [where …] [group by …] [depends: …]`，先做全角归一，再词法/语法分析；结构扫描与语法解析两条独立路径，检出项按固定优先级排序，规则书写顺序不影响结论。
- 核心规则二：**依赖成环检测**——按指标→指标边建图，用显式栈迭代 Tarjan 求强连通分量；分量大小 >1 或单点自环即判环，给出环路路径；同名指标按聚合/过滤/分组差异分三级冲突，保留行号最小的一条。
- 使用者与工作流：星野零售指标体系组 6 人，180 个指标散落 6 个 BI，同名不同定义最耗时；现在一次粘贴即得到「注册表 + 诊断隔离 + 环路径 + 同名冲突 + 影响面 + 变更单导出」。

## 功能模块

| 模块 | 使用者动作 | 入口 | 核心文件 | 测试 |
|---|---|---|---|---|
| M1 指标接入 | 粘贴 DSL 或选样例，看到逐行状态与阶段计数 | POST /api/import、页面 / | `app/modules/m1_ingest/service.py` `app/modules/m1_ingest/decode.py` `app/modules/m1_ingest/reader.py` `app/modules/m1_ingest/lines.py` | `tests/test_ingest.py` |
| M2 语法规则 | 查聚合/字段/维度登记表与文法，试解析单行 | GET /api/contract、POST /api/syntax/preview、页面 /contract | `app/modules/m2_syntax/grammar.py` `app/modules/m2_syntax/contract.py` `app/modules/m2_syntax/coerce.py` | `tests/test_syntax.py` |
| M3 解析诊断 | 看原因码分布与隔离行，导出隔离 CSV | GET /api/diagnostics、页面 /diagnostics | `app/modules/m3_diagnostics/rules.py` `app/modules/m3_diagnostics/metrics.py` `app/modules/m3_diagnostics/quarantine.py` | `tests/test_diagnostics.py` |
| M4 依赖图与环 | 看依赖图、环路径、同名冲突表 | GET /api/graph、页面 /graph | `app/modules/m4_graph/builder.py` `app/modules/m4_graph/tarjan.py` `app/modules/m4_graph/conflicts.py` | `tests/test_graph.py` `tests/test_tarjan.py` |
| M5 版本区间与批次 | 查版本台账、提交新版、拒绝陈旧提交、看批次历史 | GET /api/versions、POST /api/versions/submit、GET /api/batches、页面 /versions | `app/modules/m5_versions/intervals.py` `app/modules/m5_versions/stale.py` `app/modules/m5_versions/batches.py` | `tests/test_versions.py` |
| M6 影响面与导出 | 算影响分、核四账、导出指标/边/冲突/清单 | GET /api/impact、GET /api/reconcile、GET /api/export/*、页面 /impact | `app/modules/m6_impact/impact.py` `app/modules/m6_impact/export_out.py` `app/modules/m6_impact/json_out.py` | `tests/test_impact.py` |

端到端：`tests/test_e2e.py`（样例 → 接入 → 诊断 → 图 → 版本 → 影响面 → 导出，含 4xx 失败输入与幂等重跑）。

## 陷阱登记

| TRAP | 现象 | 朴素实现会怎样 | 本项目处理 | 处理位置 | 测试 |
|---|---|---|---|---|---|
| TRAP-02 | 样例文件头带 UTF-8 BOM | 首个指标名变成 `\ufeff销售额`，与报表引用对不上 | 解码第一步走 `utf-8-sig` 剥掉 BOM 并记 `E_BOM_STRIPPED` | `app/modules/m1_ingest/decode.py` | `tests/test_ingest.py::test_bom_stripped_before_first_name` |
| TRAP-05 | 行尾多一个分隔符、括号少一个、聚合带两个参数 | 静默截断成「能解析」，把错定义登记进注册表 | 结构扫描独立统计括号与记号，多余记号整行隔离 | `app/modules/m3_diagnostics/rules.py` | `tests/test_diagnostics.py::test_trailing_token_isolated` |
| TRAP-07 | `SUM(N/A)`、`where channel = 未知` | 哨兵被当成字段名或 0 参与统计 | 哨兵清单集中在配置，命中即隔离，不当 0 | `app/reasons.py`、`app/modules/m3_diagnostics/rules.py` | `tests/test_diagnostics.py::test_sentinel_value_isolated` |
| TRAP-14 | 同一指标名在三个 BI 里各写一遍，聚合/过滤/分组不同 | 后写覆盖，注册表里悄悄换成一个版本 | 保留行号最小的一条，其余按差异分 BLOCK/REVIEW/INFO 入冲突表并可导出 | `app/modules/m4_graph/conflicts.py` | `tests/test_graph.py::test_same_name_conflict_levels` |
| TRAP-17 | 字段改名：老定义写 `order_amount`，登记表是 `order_amt` | `KeyError` 或整行判成未注册字段 | 别名表收口在契约模块，命中即映射到登记名 | `app/modules/m2_syntax/contract.py` | `tests/test_syntax.py::test_alias_maps_renamed_field` |
| TRAP-19 | 一行同时命中「未知字段」和「多余记号」 | 按代码书写顺序取第一条，换顺序结论就变 | 八条扫描互不短路，最后按 `DIAG_ORDER` 统一排序 | `app/modules/m3_diagnostics/rules.py` | `tests/test_diagnostics.py::test_scan_order_does_not_change_result` |
| TRAP-20 | 指标互相引用成环、指标引用自身 | 递归求环爆栈，或拓扑排序死循环 | 显式栈迭代 Tarjan，单点自环单独判；环路路径可导出 | `app/modules/m4_graph/tarjan.py` | `tests/test_tarjan.py::test_long_ring_without_recursion_limit` |
| TRAP-21 | `depends` 声明了 A 但表达式没引用，或反过来 | 只核单向，血缘图缺半边 | 声明集合与实际引用集合双向比对，两侧各给原因码 | `app/modules/m4_graph/builder.py` | `tests/test_graph.py::test_one_sided_declaration_detected` |
| TRAP-25 | 同一指标 v1 `[01-01, 04-01)`、v2 `[03-15, 07-01)` | 两版同时生效，取到哪版看排序运气 | 半开区间 + 两两相交判定，重叠行标 `OVERLAP`，相邻不算重叠 | `app/modules/m5_versions/intervals.py` | `tests/test_versions.py::test_overlap_flagged` |
| TRAP-26 | 两个组同时改同一指标，后提交的版本号更小 | 后提交无条件覆盖，先改的被静默丢掉 | 版本号必须大于库中最新值，否则 409 `V_STALE_SUBMIT` | `app/modules/m5_versions/stale.py` | `tests/test_versions.py::test_stale_submit_rejected` |

## 四账

- 行账：输入行 = 注册指标行 + 隔离行 + 空行 + 注释行
- 节点账：图节点 = 指标 + 字段 + 维度 + 报表
- 边账：依赖边 = 指标→指标 + 指标→字段 + 指标→维度 + 报表→指标
- 分量账：指标节点 = 环内指标 + 非环指标
- 四账在 `GET /api/reconcile`、`/impact` 页面与 `accounts_json` 中显示，差额非 0 会显式为「不平」。

## 反例与不变量

- 反例（朴素实现会挂）：`tests/test_tarjan.py::test_long_ring_without_recursion_limit`（1200 个指标成环，递归实现撞递归上限）、`tests/test_ingest.py::test_naive_splitlines_splits_u2028`（`str.splitlines()` 在 U+2028 上多切一刀）、`tests/test_diagnostics.py::test_naive_first_error_only_loses_diagnostics`。
- 不变量：`tests/test_ingest.py::test_idempotent_rerun_adds_no_rows`（幂等）、`tests/test_impact.py::test_four_accounts_balance`（守恒）、`tests/test_tarjan.py::test_component_partition_is_total`（分量划分不重不漏）。
- 端到端：`tests/test_e2e.py::test_pipeline_from_sample_to_export`。

## 已知不覆盖

- 不做指标值计算：DSL 只解析结构、血缘与版本，不执行聚合，因此没有数据量与精度问题；这点故意收窄，避免依赖真实数仓。
- 版本台账只维护 4 个指标的 10 行样例，提交接口写同一批次的 `versions` 表，不回溯历史批次。
- 环检测按指标→指标边，不把报表或维度纳入环判定（报表是叶子节点）。
- 依赖图只按文本渲染成等宽视图，没有交互式拖拽布局；指标规模在几百条以内够用。
- 全角归一覆盖 `U+FF01`–`U+FF5E`、顿号、句号、全角空格；不处理竖排标点与半角片假名。
