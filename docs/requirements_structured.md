# 结构化需求 — Tuner-mock 造数框架

## 目标

- 构建一个可复用的 mock 数据框架，能够通过项目特定的规则扩展，以生成真实且有关联的测试数据集，供系统测试使用。

## 范围

- 核心：DDL → models、rules/plugins、generators、导出器（exporters）和期望断言（expectations）。
- 非范围：项目特定的重审业务逻辑与性能压测。

## 核心概念

- 主数据（Master data）：用于约束或作为生成种子的源表（例如商品/供应商主数据）。
- Mock 对象：需生成的表（例如 invoices、invoice_details、sales、sales_details、disclosures）。
- 规则（Rules）：可插拔逻辑，定义对象间的关联、采样与约束。
- 期望（Expectations）：可选层，将生成数据转为断言或期望记录供测试使用。

## 架构层次

- `models`：由 DDL 生成的 pydantic / ORM 模型。
- `rules`：可插拔的业务规则与关联策略。
- `generators`：实例化并建立关联的生成引擎（支持流式/批量生成）。
- `infrastructure`：DDL 解析、DB 连接器、CSV/SQL 导出器与批量导入辅助。
- `expectations`：生成断言模板或期望结果产物。

## 设计模式与技术选型

- 插件/策略（Plugin/Strategy）：将规则、期望与 DB 连接器实现为运行时插件。
- 工厂/构建器（Factory/Builder）：通过 builder 构造复杂实体（例如 invoice + details），便于模板与极端场景复用。
- 声明式配置 + Python 钩子：YAML/JSON 适合简单映射/规则，复杂逻辑由 Python 插件实现。
- 可复现的随机性：全局 seed + 确定性 id 命名空间以保证可重复输出。

## 推荐目录结构

- `ddl/` — 输入 DDL 或模式定义
- `models/` — 生成的 pydantic/ORM 模型
- `rules/` — 规则插件与示例
- `expectations/` — 断言模板
- `generators/` — 生成引擎与模板
- `infrastructure/` — DDL 解析器、导出器、导入辅助
- `cli.py`, `config.yaml`

## 技术建议

- 使用 `pydantic` 做数据校验与轻量模型
- 使用 `SQLAlchemy` 或 `sqlacodegen` 做 ORM/DDL 反射（视是否需要 DB 集成）
- 使用 `Faker` / `Factory Boy` 生成逼真假值

## 生成流程

1. 加载主数据（DB/DDL/config）。
2. 从 DDL 生成或加载 `pydantic` 模型。
3. 使用规则驱动的生成器（流式或批量）创建并建立实体间关联。
4. （可选）生成期望并导出 CSV/SQL 产物。
5. 提供批量导入脚本或测试库的导入说明。

## 测试与断言

- 每次生成可选产出 `expected.json`，描述关键断言。
- 测试策略：规则插件的单元测试；生成器+导出+导入的集成测试。

## 导出 / 导入注意事项

- 对于大规模输出，优先使用流式 CSV 或 COPY 友好的 SQL 以避免 OOM。
- 为 Postgres（`COPY`）和 MySQL（`LOAD DATA INFILE`）提供导入辅助脚本。

## 下一步（推荐最小可行产品）

1. 搭建仓库骨架与 `cli.py`。
2. 添加示例 DDL 与 `ddl2models` 脚本（生成 pydantic 模型）。
3. 实现最小生成器，能根据模板生成 invoices + invoice_details。
4. 添加示例规则插件，实现发票与主数据关联。
5. 添加 CSV 导出器与用于测试用例 2（发票→主数据关联）的系统测试脚本。

## 我可以现在实现的选项

- 搭建最小实现（文件骨架 + 示例 DDL + 演示生成器），或
- 起草简洁的 DSL（YAML）用于表达简单关联规则并给出规则接口规范。

请告知你偏好哪项以及目标 DB（Postgres / MySQL / 其它），我将据此继续落实脚手架或 DSL 草案。
