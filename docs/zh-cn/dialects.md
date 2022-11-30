## MySQL

MySQL 支持 [迁移](migrate.md) 部分中提到的所有功能。 而且以下3个版本会持续地进行测试。`5.6.35`, `5.7.26` 和 `8`。

## MariaDB

MariaDB支持[数据迁移](migrate.md)部分的全部特性。而且将会对`10.2`，`10.3`和最新版本三个版本进行持续测试。

## PostgreSQL

PostgreSQL支持[数据迁移](migrate.md)部分的全部特性，并且会对`11`，`11`，`12`，`13`，`14`五个版本进行持续测试。

## CockroachDB **(<ins>预览</ins>)**

CockroachDB在预览版本中支持，并且需要[数据集迁移引擎](#atlas-integration)。  
最近在`v21.2.11`的版本里进行了与CRDB的集成测试。

## SQLite

[Atlas](https://github.ariga/atlas) SQLite驱动支持[迁移](migrate.md)章节中提到的全部特性。 注意某些更改，比如修改列，是在一个临时表上使用[SQLite官方文档](https://www.sqlite.org/lang_altertable.html#otheralter)中描述的操作顺序完成的。

## Gremlin

Gremlin并不支持迁移与索引，**<ins>目前仅作为实验性功能</ins>**.

## TiDB **(<ins>预览</ins>)**

TiDB在预览版本中支持，并且需要 [数据集迁移引擎](#atlas-integration).  
TiDB与MySQL兼容，因此任何在MySQL上可用的特性都_应该_在TiDB上良好适用。  
已知的兼容性问题列表参见：https://docs.pingcap.com/tidb/stable/mysql-compatibility  
已对TiDB`5.4.0`, `6.0.0`版本进行测试。
