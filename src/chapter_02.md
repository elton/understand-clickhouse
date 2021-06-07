# 引擎

## 数据库引擎

您使用的所有表都是由数据库引擎所提供的

默认情况下，ClickHouse 使用自己的数据库引擎，该引擎提供可配置的表引擎和所有支持的 SQL 语法.

### Atomic 引擎（默认）

它支持非阻塞 DROP 和 RENAME TABLE 查询以及原子 EXCHANGE TABLES t1 AND t2 查询。默认情况下使用原子数据库引擎

```SQL
CREATE DATABASE test ENGINE = Atomic;
```

### 延时引擎 Lazy

在距最近一次访问间隔 expiration_time_in_seconds 时间段内，将表保存在内存中，仅适用于 `*Log` 引擎表

由于针对这类表的访问间隔较长，对保存大量小的 `*Log` 引擎表进行了优化，

```SQL
CREATE DATABASE testlazy ENGINE = Lazy(expiration_time_in_seconds);
```

### MySQL

MySQL 引擎用于将远程的 MySQL 服务器中的表映射到 ClickHouse 中，并允许您对表进行 INSERT 和 SELECT 查询，以方便您在 ClickHouse 与 MySQL 之间进行数据交换。

MySQL 数据库引擎会将对其的查询转换为 MySQL 语法并发送到 MySQL 服务器中，因此您可以执行诸如 `SHOW TABLES` 或 `SHOW CREATE TABLE` 之类的操作。

但您无法对其执行以下操作：

- RENAME
- CREATE TABLE
- ALTER

```SQL
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster]
ENGINE = MySQL('host:port', ['database' | database], 'user', 'password')
```

MySQL 数据库引擎参数

- host:port — 链接的 MySQL 地址。
- database — 链接的 MySQL 数据库。
- user — 链接的 MySQL 用户。
- password — 链接的 MySQL 用户密码。

## 表引擎

表引擎（即表的类型）决定了：

- 数据的存储方式和位置，写到哪里以及从哪里读取数据
- 支持哪些查询以及如何支持。
- 并发数据访问。
- 索引的使用（如果存在）。
- 是否可以执行多线程请求。
- 数据复制参数。

对于大多数正式的任务，应该使用 MergeTree 族中的引擎。

### 日志引擎(Log)

这些引擎是为了需要写入许多小数据量（少于一百万行）的表的场景而开发的。

这系列的引擎有：

- StripeLog
- Log
- TinyLog

#### 共同属性

- 数据存储在磁盘上。
- 写入时将数据追加在文件末尾。
- 不支持突变操作。
- 不支持索引。 这意味着 `SELECT` 在范围查询时效率不高。
- 非原子地写入数据。如果某些事情破坏了写操作，例如服务器的异常关闭，你将会得到一张包含了损坏数据的表。

#### 差异

`Log` 和 `StripeLog` 引擎支持：

- 并发访问数据的锁。`INSERT` 请求执行过程中表会被锁定，并且其他的读写数据的请求都会等待直到锁定被解除。如果没有写数据的请求，任意数量的读请求都可以并发执行。
- 并行读取数据。在读取数据时，ClickHouse 使用多线程。 每个线程处理不同的数据块。

`Log` 引擎为表中的每一列使用不同的文件。`StripeLog` 将所有的数据存储在一个文件中。因此 `StripeLog` 引擎在操作系统中使用更少的描述符，但是 `Log` 引擎提供更高的读性能。

`TinyLog` 引擎是该系列中最简单的引擎并且提供了最少的功能和最低的性能。`TinyLog` 引擎不支持并行读取和并发数据访问，并将每一列存储在不同的文件中。它比其余两种支持并行读取的引擎的读取速度更慢，并且使用了和 `Log` 引擎同样多的描述符。你可以在简单的低负载的情景下使用它。

### MergeTree

Clickhouse 中最强大的表引擎当属 MergeTree （合并树）引擎及该系列（\*MergeTree）中的其他引擎。

MergeTree 系列的引擎被设计用于插入极大量的数据到一张表当中。数据可以以数据片段的形式一个接着一个的快速写入，数据片段在后台按照一定的规则进行合并。相比在插入时不断修改（重写）已存储的数据，这种策略会高效很多。

主要特点:

- 存储的数据按主键排序。

  这使得你能够创建一个小型的稀疏索引来加快数据检索。

- 支持数据分区，如果指定了 `分区键` 的话。

  在相同数据集和相同结果集的情况下 ClickHouse 中某些带分区的操作会比普通操作更快。查询中指定了分区键时 ClickHouse 会自动截取分区数据。这也有效增加了查询性能。

- 支持数据副本。

  `ReplicatedMergeTree` 系列的表提供了数据副本功能。

- 支持数据采样。

  需要的话，你可以给表设置一个采样方法。

```SQL
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
[SETTINGS name=value, ...]
```

- ENGINE - 引擎名和参数。 ENGINE = MergeTree(). MergeTree 引擎没有参数。

- ORDER BY — 排序键。

  可以是一组列的元组或任意的表达式。 例如: ORDER BY (CounterID, EventDate) 。

  如果没有使用 PRIMARY KEY 显式的指定主键，ClickHouse 会使用排序键作为主键。

  如果不需要排序，可以使用 ORDER BY tuple().

- PARTITION BY — 分区键 。

  要按月分区，可以使用表达式 toYYYYMM(date_column) ，这里的 date_column 是一个 Date 类型的列。分区名的格式会是 "YYYYMM" 。

- PRIMARY KEY - 主键，如果要 选择与排序键不同的主键，可选。

  默认情况下主键跟排序键（由 ORDER BY 子句指定）相同。
  因此，大部分情况下不需要再专门指定一个 PRIMARY KEY 子句。

- SAMPLE BY — 用于抽样的表达式。

  如果要用抽样表达式，主键中必须包含这个表达式。例如：
  `SAMPLE BY intHash32(UserID) ORDER BY (CounterID, EventDate, intHash32(UserID))` 。

- TTL 指定行存储的持续时间并定义数据片段在硬盘和卷上的移动逻辑的规则列表，可选。

  表达式中必须存在至少一个 `Date` 或 `DateTime` 类型的列，比如：

  `TTL date + INTERVAl 1 DAY`

  规则的类型 `DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'` 指定了当满足条件（到达指定时间）时所要执行的动作：移除过期的行，还是将数据片段（如果数据片段中的所有行都满足表达式的话）移动到指定的磁盘（`TO DISK 'xxx'`） 或 卷（`TO VOLUME 'xxx'`）。默认的规则是移除（DELETE）。可以在列表中指定多个规则，但最多只能有一个 DELETE 的规则。

- SETTINGS — 控制 MergeTree 行为的额外参数：

  - `index_granularity` — 索引粒度。索引中相邻的『标记』间的数据行数。默认值，8192 。
  - `index_granularity_bytes` — 索引粒度，以字节为单位，默认值: 10Mb。如果想要仅按数据行数限制索引粒度, 请设置为 0(不建议)。
  - `enable_mixed_granularity_parts` — 是否启用通过 `index_granularity_bytes` 控制索引粒度的大小。在 19.11 版本之前, 只有 `index_granularity` 配置能够用于限制索引粒度的大小。当从具有很大的行（几十上百兆字节）的表中查询数据时候，`index_granularity_bytes` 配置能够提升 ClickHouse 的性能。如果你的表里有很大的行，可以开启这项配置来提升 SELECT 查询的性能。
  - `use_minimalistic_part_header_in_zookeeper` — 是否在 ZooKeeper 中启用最小的数据片段头 。如果设置了 `use_minimalistic_part_header_in_zookeeper=1` ，ZooKeeper 会存储更少的数据。
  - `min_merge_bytes_to_use_direct_io` — 使用直接 I/O 来操作磁盘的合并操作时要求的最小数据量。合并数据片段时，ClickHouse 会计算要被合并的所有数据的总存储空间。如果大小超过了 `min_merge_bytes_to_use_direct_io` 设置的字节数，则 ClickHouse 将使用直接 I/O 接口（O_DIRECT 选项）对磁盘读写。如果设置 `min_merge_bytes_to_use_direct_io = 0` ，则会禁用直接 I/O。默认值：`10 * 1024 * 1024 * 1024` 字节。
  - `merge_with_ttl_timeout` — TTL 合并频率的最小间隔时间，单位：秒。默认值: 86400 (1 天)。
  - `write_final_mark` — 是否启用在数据片段尾部写入最终索引标记。默认值: 1（不建议更改）。
  - `merge_max_block_size` — 在块中进行合并操作时的最大行数限制。默认值：8192
  - `storage_policy` — 存储策略。 参见 使用具有多个块的设备进行数据存储.
  - `min_bytes_for_wide_part`,`min_rows_for_wide_part` 在数据片段中可以使用 Wide 格式进行存储的最小字节数/行数。你可以不设置、只设置一个，或全都设置。参考：数据存储

#### 数据存储

表由按主键排序的数据片段（DATA PART）组成。

当数据被插入到表中时，会创建多个数据片段并按主键的字典序排序。例如，主键是 `(CounterID, Date)` 时，片段中数据首先按 `CounterID` 排序，具有相同 `CounterID` 的部分按 `Date` 排序。

数据片段可以以 `Wide` 或 `Compact` 格式存储。在 `Wide` 格式下，每一列都会在文件系统中存储为单独的文件，在 `Compact` 格式下所有列都存储在一个文件中。`Compact` 格式可以提高插入量少插入频率频繁时的性能。

数据存储格式由 `min_bytes_for_wide_part` 和 `min_rows_for_wide_part` 表引擎参数控制。如果数据片段中的字节数或行数少于相应的设置值，数据片段会以 `Compact` 格式存储，否则会以 `Wide` 格式存储。

**ClickHouse 不要求主键惟一，所以你可以插入多条具有相同主键的行。**

#### 主键选择

主键中列的数量并没有明确的限制。依据数据结构，你可以在主键包含多些或少些列。这样可以：

- 改善索引的性能。

  如果当前主键是 (a, b) ，在下列情况下添加另一个 c 列会提升性能：

  - 查询会使用 c 列作为条件
  - 很长的数据范围（ index_granularity 的数倍）里 (a, b) 都是相同的值，并且这样的情况很普遍。换言之，就是加入另一列后，可以让你的查询略过很长的数据范围。

- 改善数据压缩。

  ClickHouse 以主键排序片段数据，所以，数据的一致性越高，压缩越好。

- 在 CollapsingMergeTree 和 SummingMergeTree 引擎里进行数据合并时会提供额外的处理逻辑。

  在这种情况下，指定与主键不同的 排序键 也是有意义的。

长的主键会对插入性能和内存消耗有负面影响，但主键中额外的列并不影响 SELECT 查询的性能。

**可以使用 `ORDER BY tuple()` 语法创建没有主键的表**。在这种情况下 ClickHouse 根据数据插入的顺序存储。如果在使用 `INSERT ... SELECT` 时希望保持数据的排序，请设置 `max_insert_threads = 1`。
