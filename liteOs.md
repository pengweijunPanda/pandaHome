# LiteOS 轻量关系型数据库技术方案

## 1. 文档说明

本文描述 `liteOS_db` 当前已经实现并通过测试验证的技术方案，同时标出尚未实现或需要下游适配的部分。设计面向 LiteOS 穿戴设备，核心目标如下：

- 使用 C99，实现内存占用小于 1 MB 的关系型持久化数据库。
- 运行阶段不调用 `malloc/free`，减少 LiteOS 动态内存开销和碎片。
- 文件访问全部通过 `NB_Vfs`，降低系统调用次数并隔离操作系统差异。
- 使用标准 SQL 子集，不引入业务专用 SQL 语法。
- 针对运动明细和运动统计采用两种独立存储结构。
- API 使用 `NB_` 前缀，后续语义采用大驼峰命名。

本文以仓库当前源码为准。历史讨论中的候选方案如果未进入实现，不作为当前能力承诺。

## 2. 需求边界

### 2.1 运动明细场景

运动明细以时间戳作为 `INTEGER PRIMARY KEY`，主键严格递增写入。典型业务模型为每小时 720 条、连续保存多个小时，测试模型覆盖 72,000 条记录。

该场景要求：

- 支持 `INSERT`、`UPDATE`、`DELETE` 和 `SELECT`。
- 支持通过主键点查。
- 支持最多 5 个查询条件，以 `AND`、`OR` 和括号组合。
- 查询结果只按主键自然升序返回。
- 不支持聚合、倒序或其他字段排序。
- 插入和顺序读取优先，文件结构尽量简单。

### 2.2 运动统计场景

统计表每条记录表示一小时的统计信息，最多保存 100 条。

该场景要求：

- 支持 `INSERT`、`UPDATE`、`DELETE` 和 `SELECT`。
- 支持通过主键查询。
- 支持最多 5 个条件的 `AND`、`OR` 组合过滤。
- 支持 `SUM`、`AVG`、`MAX`、`MIN`。
- 支持最多 100 条结果的单字段 `ORDER BY ASC/DESC`。

### 2.3 测试基线

测试分为两个层次：

- 基础摸底：构造 10,000 条运动明细，通过 `WHERE` 筛选其中 720 条。
- 完整端到端：构造 72,000 条运动明细和 100 条统计记录，验证查询、聚合、排序、更新、删除、持久化重开和耗时日志。

## 3. 总体架构

```text
业务 SQL / NB_ API
        |
        v
+---------------------------+
| PreparedStmt 固定对象池   |
| 流式 Lexer + 定长 AST     |
+---------------------------+
        |
        v
+---------------------------+
| Schema Catalog / 语义绑定 |
+---------------------------+
        |
        v
+-------------------+  +-------------------+
| Time-series Store |  | Stat Store        |
| 明细顺序文件      |  | 100 条内存槽位    |
+-------------------+  +-------------------+
        |                       |
        +-----------+-----------+
                    v
              NB_Vfs 适配层
                    |
                    v
           LiteOS 文件系统实现
```

模块划分：

| 目录 | 职责 |
| --- | --- |
| `include/nb_db.h` | 对外 API、状态码、配置、VFS 和结果集接口 |
| `src/base/` | DB 生命周期、arena、PreparedStmt、绑定、执行器、ResultSet、批处理 |
| `src/sql/` | 流式词法分析和定长 AST 语法解析 |
| `src/schema/` | 固定容量 schema、行布局、语义校验和 schema 持久化 |
| `src/storage/ts/` | Time-series 顺序文件、稀疏索引、块缓存、游标和 redo journal |
| `src/storage/stat/` | Stat 内存槽位、排序、遍历和 redo journal |
| `tests/` | 单元测试、端到端测试和主机 POSIX VFS |

## 4. 内存模型

### 4.1 Arena 概念

Arena 是调用方在 `NB_DBOpen` 时提供的一整块连续内存工作区。数据库只维护当前偏移量，按对齐要求从前向后分配对象；运行期间不逐个申请或释放内存。

```c
static union {
    uint64_t alignment;
    unsigned char data[128u * 1024u];
} workArea;

NB_DBOption option;
memset(&option, 0, sizeof(option));
option.structSize = sizeof(option);
option.apiVersion = NB_API_VERSION;
option.vfs = &liteOsVfs;
option.workBuffer = workArea.data;
option.workBufferSize = sizeof(workArea.data);
```

Arena 中保存：

- `NB_DB` 对象和数据库路径。
- PreparedStmt、ResultSet 和表运行时对象池。
- SQL 文本、绑定值缓冲区和批处理快照。
- Schema catalog 及其 I/O 缓冲区。
- 表行临时缓冲区。
- Time-series 块缓存和稀疏索引。
- Stat 表的 100 个完整槽位。

`NB_DBClose` 只关闭文件、同步数据并使 DB 句柄失效，不释放调用方提供的内存。关闭成功后，调用方可以整体复用该工作区。

### 4.2 容量建议

当前 64 位主机测量结果可作为穿戴设备配置起点：

| 使用方式 | 测得最低值 | 建议工作区 |
| --- | ---: | ---: |
| 1 个 PreparedStmt、1 个 ResultSet、批量 1 条 | 34,984 B | 64 KB |
| 2 个 PreparedStmt、1 个 ResultSet、批量 16 条 | 59,088 B | 128 KB |
| 2 个 PreparedStmt、批量 100 条 | 200,944 B | 256 KB |
| 默认配置 | 161,008 B | 192 至 256 KB |

推荐穿戴设备首先使用 128 KB，并按业务字段宽度和批量大小实测。工作区不足时 `NB_DBOpen` 或懒加载表的操作返回 `NB_ERR_NOMEM`，不会退化为动态分配。

影响工作区大小的主要参数：

- `maxPreparedStmts`、`maxResultSets` 和 `maxBatchRows`。
- `maxTables`、`maxSqlLength` 和 `bindBufferSize`。
- `blockCacheSize` 和 `sparseIndexSize`。
- 表的固定 `rowSize`，特别是 `TEXT/BLOB` 最大宽度。

Time-series 典型行长为 40 B 时，槽位为 52 B；128 条块缓存约 6,656 B。72,000 条记录需要约 563 个稀疏索引项，64 位构建下每项通常为 16 B，建议索引区至少 12 KB。

Stat 典型行长为 68 B 时，单槽位为 80 B，100 条数据区约 8 KB。

### 4.3 固定池和生命周期

- PreparedStmt 和 ResultSet 均来自固定对象池。
- 池满时返回 `NB_ERR_FULL`。
- PreparedStmt 有活动 ResultSet 时不能释放。
- DB 有未释放的 PreparedStmt 或 ResultSet 时，`NB_DBClose` 返回 `NB_ERR_BUSY`。
- 同一张 Time-series 表当前只允许一个活动流式游标。
- ResultSet 返回的 `TEXT/BYTES` 指针只保证在下一次 `NB_ResultSetNext` 或释放结果集之前有效。

## 5. Schema 与记录布局

### 5.1 Schema 约束

- 默认最多 8 张表，可通过 `maxTables` 调整。
- 每表默认最多 16 列。
- 标识符最长 31 个字符。
- 主键必须是 `INTEGER PRIMARY KEY`。
- `TEXT` 和 `BLOB` 使用固定最大宽度，避免变长堆分配。
- 可使用 `VARCHAR(N)`、`BLOB(N)` 声明宽度，也可在建表后通过 `NB_PRAGMA_COLUMN_WIDTH` 设置。
- 建表后通过 `NB_PRAGMA_STORAGE_TYPE` 指定 `TIME_SERIES` 或 `STAT`。

示例：

```sql
CREATE TABLE sport_data (
    ts INTEGER PRIMARY KEY,
    steps INTEGER NOT NULL,
    heart_rate INTEGER,
    distance REAL,
    state VARCHAR(16)
);
```

```c
NB_StorageType type = NB_STORAGE_TIME_SERIES;
NB_SchemaPragma(db, "sport_data", NB_PRAGMA_STORAGE_TYPE, &type, NULL);
```

### 5.2 固定行布局

每条业务行由以下区域构成：

1. 空值位图，大小为 `(columnCount + 7) / 8`。
2. `INTEGER/REAL`：按 8 字节对齐，占 8 字节。
3. `TEXT/BYTES`：按 2 字节对齐，使用 2 字节实际长度和固定宽度内容区。

字段在创建 schema 时确定偏移和存储长度，因此记录可以原地定位，不需要解析变长页或执行内存分配。

### 5.3 Schema 文件

Schema 保存为 `<dbName>.nbs`，采用显式小端编码，不直接落盘 C 结构体。

文件组成：

```text
+----------------------+ 0
| Header, 32 bytes     |
+----------------------+ 32
| Table 0, 752 bytes   |
+----------------------+
| Table 1, 752 bytes   |
+----------------------+
| ...固定容量表项      |
+----------------------+
```

Header 主要字段：

| 偏移 | 长度 | 含义 |
| ---: | ---: | --- |
| 0 | 4 | Magic `NBS1` |
| 4 | 2 | 格式版本，当前为 1 |
| 6 | 2 | Header 长度，32 |
| 8 | 2 | 表容量 |
| 10 | 2 | 已使用表数量 |
| 12 | 4 | Payload 长度 |
| 16 | 4 | Payload CRC32 |
| 20 | 12 | 保留 |

每个表项固定 752 B，由 48 B 表信息和 16 个 44 B 列信息组成。加载时校验文件长度、版本、容量、表数量和 CRC32。

当前 schema 保存流程为完整文件覆盖、截断和 `sync`。CRC 能检测损坏，但 schema 尚未使用双副本或 journal；掉电发生在 schema 重写期间时，可能返回 `NB_ERR_CORRUPT`，这是后续可靠性增强项。

## 6. Time-series 存储设计

### 6.1 设计取舍

Time-series 表采用固定槽位顺序文件。它利用时间戳主键递增这一业务特性，省去 B-Tree 页分裂、通用空闲页管理和大规模内存索引。

优势：

- `INSERT` 是单槽位追加写。
- 主键顺序即物理顺序，范围读取连续。
- 稀疏索引只保存每个块的首主键和首槽位。
- 删除使用墓碑，不移动后续记录。

代价：

- 新主键必须严格大于历史最后主键，否则返回 `NB_ERR_ORDER`。
- 删除不会立即回收文件空间。
- 只支持主键自然升序，不支持通用排序和聚合。

### 6.2 数据文件格式

数据文件名为 `<dbName>.<tableName>.tsd`。

```text
+--------------------------+ 0
| File Header, 32 bytes    |
+--------------------------+ 32
| Slot 0 Header, 12 bytes  |
| Slot 0 Row, rowSize      |
+--------------------------+
| Slot 1 Header, 12 bytes  |
| Slot 1 Row, rowSize      |
+--------------------------+
| ...                      |
+--------------------------+
```

文件头字段：

| 偏移 | 长度 | 含义 |
| ---: | ---: | --- |
| 0 | 4 | Magic `NBT1` |
| 4 | 2 | 格式版本 |
| 6 | 2 | Header 长度，32 |
| 8 | 2 | `rowSize` |
| 10 | 2 | `slotSize = 12 + rowSize` |
| 12 | 4 | Schema 版本 |
| 16 | 12 | 保留 |
| 28 | 4 | 前 28 B 的 CRC32 |

槽位头字段：

| 偏移 | 长度 | 含义 |
| ---: | ---: | --- |
| 0 | 8 | 主键时间戳 |
| 8 | 1 | 状态：1 为有效，2 为已删除 |
| 9 | 1 | 保留 |
| 10 | 2 | 槽位头前 10 B 与 row payload 的 CRC16 |

打开文件时根据文件长度计算槽位数。末尾不足一个完整槽位的部分会被截断，避免一次中断的追加写破坏此前完整记录。

### 6.3 稀疏索引与块缓存

默认每 128 个槽位组成一个逻辑块。稀疏索引项为：

```c
typedef struct {
    uint64_t firstPk;
    uint32_t firstSlot;
} NB_TsIndexEntry;
```

打开表时重建稀疏索引。主键点查先在索引中定位候选块，再在块内搜索；范围查询定位起始块后顺序读取。块缓存由 arena 提供，大小通过 `blockCacheSize` 控制。

### 6.4 写入、更新和删除

`INSERT`：

- 校验主键严格递增。
- 编码完整槽位并追加到文件尾部。
- 达到新的块边界时增加稀疏索引项。
- 根据 durability 策略决定何时 `sync`。

`UPDATE`：

- SQL 条件必须包含可安全提取的主键等值条件。
- 不允许修改主键。
- 定位槽位后覆盖完整槽位，并重新计算 CRC16。

`DELETE`：

- SQL 条件必须包含可安全提取的主键等值条件。
- 将槽位状态改为墓碑，保留主键和物理位置。
- 后续点查和扫描跳过墓碑。

当前没有实现在线压缩。枚举值 `NB_PRAGMA_COMPACT` 已预留，但调用会返回 `NB_ERR_UNSUPPORTED`。

### 6.5 查询路径

Time-series SELECT 使用流式 ResultSet：

1. 从主键条件推导尽可能小的扫描区间。
2. 通过稀疏索引定位起始块。
3. 块读取到预分配缓存。
4. 跳过墓碑并执行最多 5 个谓词。
5. 命中后直接向调用方暴露当前行，不缓存完整结果集。

因此返回 720 条记录时，额外内存不会随结果条数线性增长。

## 7. Stat 存储设计

### 7.1 内存驻留模型

Stat 表最多 100 条。打开时将所有完整槽位一次性读入 arena，查询、过滤、聚合和排序阶段不再读取数据文件。

该设计将工作量限制为固定上界：

- 主键查询最坏扫描 100 条。
- 最多 5 个谓词，每个查询最坏评估 500 次条件。
- 聚合最多处理 100 个数值。
- 排序使用稳定插入排序，最坏比较次数有固定上界。

对 100 条数据而言，线性扫描比维护通用 B-Tree 更简单，也避免额外文件 I/O 和内存结构。

### 7.2 文件格式

数据文件名为 `<dbName>.<tableName>.stt`，格式与 Time-series 一致：32 B 文件头、12 B 槽位头和固定 `rowSize` payload。Magic 为 `NBS1` 的 Stat 数据格式标识，文件头和槽位分别使用 CRC32、CRC16 校验。

Stat 内存中保留文件槽位，包括墓碑。`recordCount` 只统计有效记录，最大为 100。

### 7.3 写入和查询

- 插入先复用墓碑槽位；没有墓碑时追加新槽位。
- 主键重复返回 `NB_ERR_DUPLICATE`。
- 有效记录已达 100 条时返回 `NB_ERR_FULL`。
- 主键点查、更新和删除使用最多 100 条的线性定位。
- 查询先保存命中槽位编号，再按需要执行聚合或排序。
- 单字段排序使用稳定插入排序，支持 `ASC` 和 `DESC`。

## 8. Redo Journal 与持久化

### 8.1 Journal 文件

Time-series 和 Stat 的原地 `UPDATE/DELETE` 使用单记录 redo journal：

- Time-series：`<dbName>.<tableName>.tsj`
- Stat：`<dbName>.<tableName>.stj`

Journal 格式：

```text
+---------------------------+ 0
| Journal Header, 24 bytes  |
+---------------------------+ 24
| 完整的新槽位内容          |
+---------------------------+
```

Header 保存 magic、版本、header 长度、目标文件偏移、槽位长度和槽位 payload 的 CRC32。

### 8.2 写入顺序

原地更新采用以下顺序：

1. 将目标偏移和新槽位写入 journal。
2. `sync` journal。
3. 将新槽位写入数据文件。
4. `sync` 数据文件。
5. 将 journal 截断为 0，并再次 `sync`。

打开表时，如果发现长度完整且校验通过的 journal，则把新槽位幂等重放到数据文件，再清空 journal。长度错误、字段不匹配或 CRC 错误返回 `NB_ERR_CORRUPT`。

该 journal 只覆盖单槽位原地更新和删除，不是通用事务日志。批处理仍是逐行执行，不保证全批原子性。

### 8.3 同步策略

公开策略：

| 策略 | 行为 |
| --- | --- |
| `NB_DURABILITY_MANUAL` | 普通追加写后不主动同步，由 `NB_PRAGMA_FLUSH` 或关闭触发 |
| `NB_DURABILITY_SYNC_EACH_WRITE` | 每次普通写入后同步 |
| `NB_DURABILITY_SYNC_INTERVAL` | 累积指定记录数后同步 |

`syncIntervalRecords` 默认 100，必须大于 0。为了保证原地覆盖可恢复，当前 `UPDATE/DELETE` 的 journal 路径固定执行 journal 和数据文件同步，其强度高于普通追加写策略。

## 9. SQL 解析器

### 9.1 解析模型

SQL 模块采用流式 lexer 和定长 AST：

- lexer 按当前位置读取 token，不构造动态 token 数组。
- AST 嵌入 PreparedStmt，不动态创建语法节点。
- SQL 文本复制到 PreparedStmt 的固定缓冲区。
- 标识符和字符串在 AST 中使用偏移和长度引用 SQL 缓冲区。
- 解析错误通过 `errStart` 返回 SQL 字符偏移。

主要限制：

| 项目 | 上限 |
| --- | ---: |
| SQL 长度 | 默认 1,024 B |
| 绑定参数 | 32 个 |
| 表字段 | 默认 16 个 |
| WHERE 谓词 | 5 个 |
| WHERE 不同字段 | 5 个 |
| 布尔节点 | 9 个 |
| 括号嵌套 | 8 层 |

### 9.2 支持的 SQL 子集

支持：

```sql
CREATE TABLE table_name (...);
INSERT INTO table_name (...) VALUES (...);
UPDATE table_name SET ... WHERE ...;
DELETE FROM table_name WHERE ...;
SELECT ... FROM table_name WHERE ... ORDER BY column ASC|DESC LIMIT n;
```

表达式能力：

- 比较：`=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`。
- 逻辑：`AND`, `OR` 和括号。
- 值：整数、实数、字符串、`NULL` 和绑定参数。
- 参数：匿名 `?` 以及编号参数。
- 聚合：`SUM`、`AVG`、`MAX`、`MIN`。
- 排序：一个 `ORDER BY` 字段。
- 支持 SQL 注释和末尾分号。

暂不支持：

- `JOIN`、`GROUP BY`、`HAVING`。
- 子查询、表达式计算和用户函数。
- 多行 `VALUES`。
- 通用事务、回滚和隔离级别。
- 多字段 `ORDER BY`。

### 9.3 存储语义检查

`NB_Prepare` 在语法解析后进行 schema 绑定和存储约束检查：

- 表和列必须存在，字段类型必须兼容。
- Time-series 不允许聚合。
- Time-series 不允许非主键排序或主键倒序。
- Time-series 主键不能被 `UPDATE` 修改。
- Time-series `UPDATE/DELETE` 必须能从条件中安全提取主键等值约束。
- Stat 才允许通用 `ORDER BY` 和聚合。
- WHERE 谓词或字段数超过 5 返回范围错误。

## 10. API 设计

### 10.1 命名规范

所有公开接口使用 `NB_` 前缀，后续单词使用大驼峰，例如：

```c
int NB_DBOpen(...);
int NB_Prepare(...);
int NB_ExecuteQuery(...);
int NB_ResultSetGetInt64(...);
```

绑定参数下标从 1 开始；结果集字段下标从 0 开始。

### 10.2 Base API

```c
int NB_DBOpen(const char *dbName, NB_DBOption *dbOption,
              unsigned int flags, NB_DB **db);
int NB_DBClose(NB_DB *db, unsigned int flags);
int NB_DBPragma(NB_DB *db, NB_PragmaOptionCmd cmd, const void *data,
                NB_ResultSet **nbResultSet);
int NB_SchemaPragma(NB_DB *db, const char *schemaName,
                    NB_PragmaOptionCmd cmd, const void *data,
                    NB_ResultSet **nbResultSet);
int NB_DBGetLastError(NB_DB *db, const char **message, int *sqlOffset);
```

当前 `NB_DBPragma` 已实现：

- `NB_PRAGMA_FLUSH`
- `NB_PRAGMA_DURABILITY`
- `NB_PRAGMA_SYNC_INTERVAL`

当前 `NB_SchemaPragma` 已实现：

- `NB_PRAGMA_STORAGE_TYPE`
- `NB_PRAGMA_COLUMN_WIDTH`

`NB_PRAGMA_MEMORY_USAGE` 和 `NB_PRAGMA_COMPACT` 是预留枚举，当前返回 `NB_ERR_UNSUPPORTED`。

### 10.3 SQL API

```c
int NB_Prepare(NB_DB *db, const char *sql, unsigned int flags,
               NB_PreparedStmt **preparedStmt, int *errStart);
int NB_FreePreparedStmt(NB_PreparedStmt *preparedStmt);
int NB_ExecuteUpdate(NB_DB *db, NB_PreparedStmt *preparedStmt,
                     unsigned int flags, int *updatedCount);
int NB_ExecuteQuery(NB_DB *db, NB_PreparedStmt *preparedStmt,
                    unsigned int flags, NB_ResultSet **resultSet);

int NB_AddBatch(NB_PreparedStmt *preparedStmt);
int NB_ExecuteBatch(NB_DB *db, NB_PreparedStmt *preparedStmt,
                    unsigned int flags, BatchExecuteRet **batchExecuteRet);
void NB_FreeBatchExecuteRet(BatchExecuteRet *batchExecuteRet);
```

`errStart` 和 `updatedCount` 均为输出指针。批处理保存绑定值快照，执行结果包含总数、成功数、失败数和每一行的状态码。批处理逐行执行，不具备事务原子性。

### 10.4 参数绑定与结果集

```c
int NB_StmtBind(NB_PreparedStmt *stmt, int index, int type,
                const void *value, int valueLen);
int NB_StmtBindNull(NB_PreparedStmt *stmt, int index);
int NB_StmtBindInt64(NB_PreparedStmt *stmt, int index, long long value);
int NB_StmtBindReal(NB_PreparedStmt *stmt, int index, double value);
int NB_StmtBindText(NB_PreparedStmt *stmt, int index, const char *value);
int NB_StmtBindBytes(NB_PreparedStmt *stmt, int index,
                     const unsigned char *value, int valueLen);
int NB_StmtClearBindings(NB_PreparedStmt *stmt);
```

ResultSet 提供 `Next`、列数量、列名、类型、空值判断和各类型 getter。遍历结束返回 `NB_DONE`，成功返回 `NB_OK`。

### 10.5 状态码原则

状态码为固定整数，不依赖 `errno`：

- `NB_OK`：成功。
- `NB_DONE`：结果集遍历结束。
- `NB_ERR_IO`：VFS 操作失败。
- `NB_ERR_CORRUPT`：文件格式或校验失败。
- `NB_ERR_FULL`：固定池、Stat 表或批处理容量已满。
- `NB_ERR_DUPLICATE`：主键重复。
- `NB_ERR_ORDER`：Time-series 主键未递增。
- `NB_ERR_NOMEM`：arena 空间不足。
- `NB_ERR_SQL`、`NB_ERR_SCHEMA`、`NB_ERR_TYPE`、`NB_ERR_RANGE`：解析、schema、类型或上限错误。

详细错误信息和 SQL 偏移可通过 `NB_DBGetLastError` 获取。

## 11. NB_DB 与已有项目兼容

### 11.1 当前内部结构

`NB_DB` 在轻量数据库状态之前保留了已有项目需要的兼容字段：

```c
#define NB_KVSET_NUM 4u

typedef struct NB_MetaData {
    List metaIdxCache;
    uint8_t idxCacheDirty;
    uint8_t curSchemaCount;
    uint8_t curTableCount;
    uint8_t memorypwrsiscount;
} NB_MetaData;

struct NB_DB {
    uint32_t magic;

    OsEncrypt *osEncrypt;
    char *dbName;
    Records *meta;
    Records *memIdxData;
    NB_MetaData metaData;
    List kvSetList[NB_KVSET_NUM];
    Store *store[NB_KVSET_NUM];
    LRU *schemaLru;
    Store *memStore;
    volatile int refCount;
    EvictionMemoryNode *evictionNode;

    NB_Arena arena;
    NB_DBOption option;
    NB_PreparedStmt *statements;
    NB_ResultSet *resultSets;
    NB_TableRuntime *tableRuntimes;
    NB_SchemaCatalog schemaCatalog;
    char *schemaPath;
    unsigned char *schemaIoBuffer;
    uint32_t schemaIoBufferSize;
    char lastError[NB_ERROR_MESSAGE_SIZE];
    int lastSqlOffset;
};
```

独立版本只保存 `OsEncrypt`、`Records`、`Store`、`LRU` 和 `EvictionMemoryNode` 指针，不访问其成员。兼容指针初始为 `NULL`，其指向对象视为下游借用对象，不由 `NB_DBClose` 释放。

`refCount` 当前初始化为 1。`volatile` 只能限制编译器优化，不能保证并发原子性；多任务共享 DB 时应由下游接入 LiteOS 原子操作或互斥锁。

### 11.2 List ABI

独立构建提供以下兼容定义：

```c
typedef struct ListNode {
    struct ListNode *pre;
    struct ListNode *next;
    void *value;
} ListNode;

typedef struct List {
    ListNode *head;
    ListNode *tail;
    void *(*dup)(void *ptr);
    void (*finalize)(void *ptr);
    void (*free)(void *ptr);
    void (*cmp)(void *ptr, void *key);
    unsigned long len;
} List;
```

现有工程如果已有权威类型定义，可在编译时定义 `NB_LEGACY_TYPES_HEADER`，使 `src/base/nb_legacy_compat.h` 包含下游头文件，避免重复声明和 ABI 偏差。

### 11.3 Store 与 Records 接入

下游补充的结构方向如下：

```c
struct Store {
    List recordsList;
    List wpCallBackArgsList;
    Page *page;
    StoreType type;
    uint32_t pageSize;
    Pager *pager;
    const char *fileName;
    OsEncrypt *osEncrypt;
    int (*flush)(struct Store *, int);
    /* 后续字段 */
};

struct Records {
    Store *store;
    int recordsId;
    RecordsType recordsType;
    struct AM *am;
    int rootPageId;
    ItemCompare comparator;
};
```

这些定义依赖下游的 `Page`、`Pager`、`StoreType`、`RecordsType`、`AM` 和 `ItemCompare`。当前核心不复制这些不完整类型，只通过指针兼容。正式集成前需要以已有项目权威头文件为准确认：

- `List.cmp` 的返回类型和语义。
- `uint32` 与标准 `uint32_t` 的映射。
- `Store/Records` 的完整布局和编译对齐规则。
- 外部对象的所有权、初始化和释放顺序。
- 加密层与 `NB_Vfs` 的职责边界。

## 12. VFS 与 LiteOS 适配

### 12.1 VFS 接口

核心不直接调用 POSIX、LiteOS 或 macOS 文件 API，而是依赖：

```c
typedef struct {
    int (*open)(const char *path, unsigned int flags, NB_File *file);
    int (*close)(NB_File file);
    int (*readAt)(NB_File file, uint64_t offset, void *buffer,
                  unsigned int length, unsigned int *readLength);
    int (*writeAt)(NB_File file, uint64_t offset, const void *buffer,
                   unsigned int length);
    int (*sync)(NB_File file);
    int (*size)(NB_File file, uint64_t *fileSize);
    int (*truncate)(NB_File file, uint64_t fileSize);
} NB_Vfs;
```

LiteOS 适配层应保证：

- `readAt/writeAt` 使用显式偏移，不依赖共享文件游标。
- `writeAt` 成功表示指定长度已经全部写入，否则返回非 0。
- `readAt` 通过 `readLength` 区分 EOF 和完整读取。
- `sync` 提供目标文件系统能够支持的最强持久化语义。
- `truncate` 能处理尾部残缺恢复和 journal 清空。
- VFS 自身避免每次 I/O 动态分配临时对象。

### 12.2 平台约定

Linux/POSIX 是默认主机编译和测试环境。工程遵循以下规则：

- 不使用 `_DARWIN_C_SOURCE`、`__APPLE__`、Mach API、kqueue、`F_FULLFSYNC`、macOS framework 或 macOS 专属路径。
- POSIX 适配层需要 `pread`、`pwrite`、`ftruncate` 等声明时使用 `_POSIX_C_SOURCE 200809L`。
- 核心数据库保持平台中立，平台差异只能出现在 `NB_Vfs` 适配层。
- Shell 脚本兼容 POSIX `sh` 和常见 Linux 工具。
- 使用 C99，并在 GCC 或 Clang 兼容编译器下开启 `-Wall -Wextra -Werror -pedantic`。

`tests/nb_posix_vfs.c` 只用于 Linux/POSIX 主机测试。产品集成时应替换为 LiteOS VFS。

## 13. I/O 优化策略

针对 LiteOS `malloc` 和文件 I/O 较慢的特点，当前实现采用：

- 所有核心对象由 arena 一次分配。
- Time-series 插入只追加一个固定槽位。
- 范围查询按块连续读取，而不是逐字段读取。
- 稀疏索引只为每个块保存一个入口。
- Stat 打开后全量驻留内存，查询阶段零文件读取。
- 行布局和文件字段定长，避免序列化临时对象。
- 批处理复用 PreparedStmt 和绑定缓冲区。
- 支持间隔同步，将普通追加写的多个 `sync` 合并。

性能参数需要在目标 LiteOS 文件系统上调优，尤其是块大小、同步间隔、单次批量行数和文件系统最小写入粒度。

## 14. 并发模型

当前实现按单任务或由调用方串行化访问设计：

- 核心内部没有全局锁或表锁。
- 同一个 DB 句柄不应被多个任务无锁并发调用。
- 单表只维护一个活动 Time-series 游标标记。
- `volatile refCount` 不提供线程安全。

需要并发时，建议首先在 DB 句柄外部增加 LiteOS mutex，将一次公开 API 调用作为最小互斥范围。后续如需读写并发，再单独设计锁粒度和 ResultSet 生命周期。

## 15. 测试与验证

### 15.1 测试组

CTest 注册 7 个独立测试组：

| 测试 | 验证内容 |
| --- | --- |
| `nb_arena` | 对齐分配、容量边界、无动态分配工作区 |
| `nb_prepared` | PreparedStmt 池、参数绑定、批处理快照和生命周期 |
| `nb_schema` | CREATE TABLE、字段布局、pragma、schema 保存与重开 |
| `nb_sql_execution` | 72,000 条明细和 100 条统计的 SQL 端到端行为 |
| `nb_sql_parser` | SQL 语法、5 条件、AND/OR、聚合、排序和错误偏移 |
| `nb_stat_store` | 100 条上限、墓碑复用、更新删除、重开和 I/O 次数 |
| `nb_ts_store` | 10,000 条追加、点查、范围读、过滤、更新删除和恢复 |

关键验证：

- 10,000 条预置明细中过滤得到 720 条。
- 完整模型写入 72,000 条明细和 100 条统计。
- 五条件查询返回 720 条，结果保持主键升序。
- 主键点查、更新、删除及关闭重开后结果正确。
- `SUM/AVG/MAX/MIN` 结果正确。
- 100 条数据可降序排序。
- `TEXT/BYTES` 编码和读取正确。
- 尾部残缺文件能够恢复到最后一个完整槽位。
- 查询日志输出结果数量及两位小数的毫秒耗时。

### 15.2 构建和执行

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
ctest --test-dir build --output-on-failure
```

也可执行项目脚本：

```sh
./tut.sh
```

测试程序支持单独运行测试组：

```sh
./build/nb_db_tests sql_execution
./build/nb_db_tests ts_store
```

## 16. 当前完成状态

已完成并有测试覆盖：

- Arena 和固定对象池。
- CREATE TABLE 及 schema 持久化。
- SQL lexer、parser、语义绑定和参数绑定。
- INSERT、UPDATE、DELETE、SELECT。
- 最多 5 条件的 `AND/OR` 过滤。
- Time-series 主键点查、范围扫描、自然升序和稀疏索引。
- Stat 最多 100 条、聚合和单字段排序。
- Batch 执行及逐行结果。
- 数据文件头和槽位校验。
- UPDATE/DELETE 单槽位 redo journal 与重放。
- 三种 durability 策略。
- 7 组主机自动化测试及毫秒级查询耗时日志。
- Linux/POSIX 主机约定和严格 C99 编译。

需要下游或后续版本完成：

- LiteOS 原生 `NB_Vfs` 实现及目标设备实测。
- 加密对象 `OsEncrypt` 与文件 I/O 的集成。
- 已有 `Store/Records/Pager/AM` 完整类型和生命周期接入。
- Schema 写入的双副本或 journal 保护。
- `NB_PRAGMA_MEMORY_USAGE` 和在线 `NB_PRAGMA_COMPACT`。
- 多任务并发控制。
- 通用事务、JOIN、GROUP BY 等不在当前轻量目标内的 SQL 能力。

## 17. 建议的 LiteOS 产品配置

面向“两张业务表、单任务数据库访问”的穿戴设备，建议从以下配置开始：

```c
NB_DBOption option;
memset(&option, 0, sizeof(option));
option.structSize = sizeof(option);
option.apiVersion = NB_API_VERSION;
option.vfs = &liteOsVfs;
option.workBuffer = workArea.data;
option.workBufferSize = sizeof(workArea.data); /* 建议先使用 128 KB */
option.maxTables = 2;
option.maxPreparedStmts = 2;
option.maxResultSets = 1;
option.maxBatchRows = 16;
option.maxColumns = 16;
option.maxSqlLength = 1024;
option.bindBufferSize = 256;
option.timeSeriesBlockSlots = 128;
option.blockCacheSize = 8u * 1024u;
option.sparseIndexSize = 12u * 1024u;
option.durability = NB_DURABILITY_SYNC_INTERVAL;
option.syncIntervalRecords = 100;
```

最终参数应根据真实表结构重新计算，并在设备上记录以下指标：

- `NB_DBOpen` 峰值工作区使用量。
- 720 条查询的总耗时、数据文件读取次数和读取字节数。
- 批量插入 720 条的耗时和 `sync` 次数。
- 掉电恢复时间及 journal 重放结果。
- Flash 写放大和文件增长速度。

在满足掉电可靠性要求的前提下，再调整 `timeSeriesBlockSlots` 和 `syncIntervalRecords`，以取得读取延迟、写入吞吐和 Flash 寿命之间的平衡。
