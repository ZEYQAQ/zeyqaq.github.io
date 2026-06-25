在 PostgreSQL 中，当一张表的数据量变得非常巨大时（比如业务日志表，动辄几十 GB 甚至上百 GB），单表查询和维护性能就会急剧下降。为了解决这个问题，Postgres 引入了分区表（Partitioning）机制。

**主表（父表/Parent Table）：** 这是一个“逻辑表”。它不直接存储数据（当然它是真实的物理表，有自己的OID、占用系统文件），主要用来定义表的结构（有哪些字段、什么类型）和分区规则（比如按月分区）。写 SQL 语句时（例如 `SELECT * FROM kwork_business_log.agent_business_api_call_log_v1`），直接查的就是主表。

**子表（分区表/Partition Table）：** 这是底层的“物理表”。PostgreSQL 会根据设定的规则，把数据切碎存到不同的子表里。

当系统产生一条 `2026-05-10` 的日志时，Postgres 会自动把它塞进 `..._log_v1_202605` 这张子表里。

当查询主表时，Postgres 会非常聪明地只去对应的子表里捞数据，这就大大提高了查询速度。



 DBeaver 客户端自身的 UI 默认只能看到大表，看不到子表。

如何在 DBeaver 里看子表？

1、

<img width="1920" height="1080" alt="Image" src="https://github.com/user-attachments/assets/8bc8b9e9-9535-4187-a354-0b89e425fe93" />

2、

全局设置路径是：`DBeaver` -> `首选项 (Preferences)` -> `用户界面 (User Interface)` -> `数据编辑器 (Data Editor)` 或者在 `连接类型` 中选择显示分区。



Psql中查看：

```
# 查看所有 schema 下的表（包括主表和普通表，但默认不展开子表）
\dt *.*

# 查看所有 schema 下的表，并包含详细信息（在某些 PG 版本中可看到分区信息）
\dt+ *.*
```

