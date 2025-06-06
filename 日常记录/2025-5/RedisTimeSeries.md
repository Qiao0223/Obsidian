RedisTimeSeries 是 Redis 官方推出的一个模块，专为高效处理时间序列数据而设计。它通过引入专用的数据结构和命令，简化了时序数据的存储、查询和聚合操作，适用于物联网（IoT）、监控系统、金融数据分析等场景。
# 1. 核心功能与特点

- **高效的数据写入与读取**：支持大规模数据的快速插入（如每秒百万级写入）和低延迟读取，适合高并发场景。
- **时间范围查询与聚合**：提供 `TS.RANGE`、`TS.MRANGE` 等命令，支持按时间范围查询，并可进行多种聚合操作，如 `AVG`、`MIN`、`MAX`、`SUM` 等。
- **数据压缩与保留策略**：支持数据压缩存储，节省内存空间；可配置数据的保留期限，自动删除过期数据，便于管理。
- **标签与二级索引**：每个时间序列可以设置标签（key-value 对），便于通过标签进行筛选和查询，支持 `TS.MGET`、`TS.QUERYINDEX` 等命令。
- **自动聚合规则**：通过 `TS.CREATERULE` 命令，可以定义自动聚合规则，实现数据的降采样和多粒度存储。

# 2. 基本数据结构

- **样本（Sample）**：由时间戳（64 位整数）和数值（64 位浮点数）组成。
- **块（Chunk）**：存储一组连续时间戳的样本数据，支持压缩和非压缩两种存储方式。
- **标签（Label）**：用于标识时间序列的属性，支持按标签进行查询和筛选。
- **规则（Rule）**：定义数据的自动聚合策略，实现数据的降采样和多粒度存储。