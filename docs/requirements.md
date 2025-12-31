# 造数

## 造数平台处理对象

- 主数据:从某个表中读取主数据，作为mock的范围
- mock数据对象:多个需要被mock的数据表对象
- mock数据关联规则:可配置的关联规则，使得符合业务要求
- mock数据结果期望:根据匹配规则和生成的mock对象，列出期望列表(或sql自断言)


## 设计选择

- 根据期望->读取规则->生成对应mock对象()
- 根据mock->读取规则->生成期望(#TODO:这里等于重新写了一遍业务系统逻辑？自己验证自己？)
- 看起来这里无论如何都有业务？



## 设计思路

'''code
Q:
我希望有的是一个比较通用的造数平台或框架，但又能扩展一些业务规则，或许需要一些高级的程序设计模式
1. 形式上，可以是一个造数框架，有着特定的文件目录结构，如./ddl下放置着要被造数的对象的表设计文件，他们可以被框架中的scripts转换成pydantic的数据模型，放在./models下；还可以设计一些DSL语言来标记pydantic模型的字段的关联关系，数据规则
2. 框架应分层如有个infrastructure层，可以读取数据库中的数据转换到pydantic模型，如models层定义了产品主数据的数据模型，rules层控制models之间字段的关联关系，infrastructure层可以让框架配置某个数据库链接的某个表作为某个主数据的models层来源
这样一来，测试工程师可以去自定义models层，然后灵活写py代码来配置rules规则来生成数据
3. 造数框架解耦业务，如生成期望断言更应该是每个不同项目的不同业务系统中的代码，这一部分应该由rules层管理，或开辟一个新的expectation层来管理这些逻辑
以上想法是否可行？有无成功经验？或有更好的实践？
---
A:
结论：可行，而且这是工程化且可扩展的好做法。要点与可落地建议如下（简短、面向实现）：

可行性: 可行。把 DDL → pydantic 模型、把规则抽象为可插拔的 rules/expectation 层、并提供 infrastructure 连接器，是合理且常见的方案。许多团队内部有类似平台（用 Factory/ Faker/内部 generator 构建复杂关系数据）。

高层架构（分层）:

models: 从 ./ddl 或 DB 反射/代码生成出 pydantic/ORM 模型。
rules: 由测试工程师编写的业务规则（字段关联、匹配策略、分配策略等）。规则以可注册函数或小 DSL 表达。
expectation (可选独立层): 负责把生成数据映射到期望审计结果（断言模板），每个项目可实现自己的插件。
infrastructure: 负责 DB 读写、DDL 解析、CSV/SQL 导出、批量导入、与触发 API 的集成。
generators/engine: 负责生成流程（seed 控制、实体生成、关联建立、后处理与期望计算）。
cli/web ui: 配置运行、选择规模档、启动导入/触发。
建议的目录结构 (示例)

./ddl/
./models/
./rules/
./expectations/
./infrastructure/
./generators/
cli.py, config.yaml
关键设计模式与技术选型

插件/策略模式：规则/期望/DB connector 作为插件实现，运行时注入。
工厂 + Builder：构造复杂对象（order、invoice）时使用 builder，方便复用极端场景模板。
DSL 或 声明式配置：用 YAML/JSON 表达字段约束与关联（简单规则）；对复杂逻辑允许 Python rule 插件。
Pydantic + SQLAlchemy：用 SQLAlchemy 反射或 sqlacodegen 生成 ORM，再映射到 pydantic（或直接用 pydantic-sqlalchemy）。
可复现随机：全局 seed + 命名空间 UUID 保证生成可重放。
规则引擎轻量实现：用可组合的函数和元数据替代重型引擎（如 Drools），便于调试与版本控制。
生成流程（推荐）

Load 主数据来源（DB/DDL/config）。
Instantiate models（pydantic validation）。
Entity generation（按档位 batch/stream）。
Link generation（执行 rules，建立 N:M 关系）。
Expectation derivation（rules/expectation 层产出期望断言）。
Export（CSV/SQL/COPY 文件）并提供 bulk-import 脚本或直接写入 DB。
Optionally trigger 审计接口并收集结果做断言比对。
断言/可测试性

把每个生成批次的 expected 存为 JSON；运行测试后自动比对，断言失败产生差异报告。
支持抽样或全量比对（按规模选择）。
扩展与项目适配

规则用 Python 编写时，团队可根据项目轻易替换 rules 或 expectation 插件。
提供示例 rule 与模板（如时间窗匹配、随机 N:M、极端聚合样本）。
性能与导入注意

大量数据时采用流式生成并写文件（避免内存爆炸）。
提示：导入前禁用索引/外键、按批次提交、使用 DB 的 bulk loader（Postgres COPY、MySQL LOAD DATA）。
生产极端 case（单 order 多万行）时，确保生成器和导入脚本能分块处理。
可借鉴的 OSS / 工具

Faker / Factory Boy / Mimesis（数据生成）
Mockaroo（概念参考）
WireMock / Mountebank（若需要 mock HTTP）
k6 / Locust（性能驱动）
SQLAlchemy / pydantic / pydantic-sqlalchemy（模型层）
最佳实践要点

将业务断言放在 expectation 层或规则插件里，框架只提供运行时与数据生命周期管理。
小步迭代：先实现 DDL→models→简单 rules→CSV 输出，再逐步加复杂规则与期望插件。
强制可复现（seed）、可观察（生成元数据、日志）与可回放（保存生成配置）。
为规则/expectation 编写单元测试与示例数据集，确保业务正确性不会随性能优化被破坏。
下一步我可以：

在工作区为你脚手架一个最小实现（目录结构 + DDL->pydantic 演示 + 一个 sample rule + generator CLI），并生成小批样例数据；
或先给出一个简短 DSL 语法草案（YAML 示例）和一个规则接口标准。
你希望我先做哪一项？另外告诉我目标 DB（Postgres / MySQL / 其它），我会据此生成对应的导入示例。

'''