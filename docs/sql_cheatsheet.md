# SQL 助手

本页面提供了一些关于如何查询您的 SQLite 数据库的帮助信息。

!!! Tip "其他数据库系统"
    要使用其他数据库系统（如 PostgreSQL 或 MariaDB），您可以使用相同的查询语句，但需要使用相应数据库系统的客户端。[点击此处](advanced-setup.md#use-a-different-database-system)了解如何使用 Freqtrade 设置不同的数据库系统。

!!! Warning
    如果您不熟悉 SQL，在数据库上运行查询时应格外小心。  
    在运行任何查询之前，请务必确保已对数据库进行备份。

## 安装 sqlite3

Sqlite3 是一个基于终端的 SQLite 应用程序。
如果您觉得使用图形化数据库编辑器更舒适，也可以随意使用 SqliteBrowser 等工具。

### Ubuntu/Debian 安装

```bash
sudo apt-get install sqlite3
```

### 通过 Docker 使用 sqlite3

Freqtrade Docker 镜像已包含 sqlite3，因此您无需在主机系统上安装任何软件即可编辑数据库。

``` bash
docker compose exec freqtrade /bin/bash
sqlite3 <database-file>.sqlite
```

## 打开数据库

```bash
sqlite3
.open <filepath>
```

## 表结构

### 列出数据表

```bash
.tables
```

### 显示表结构

```bash
.schema <table_name>
```

### 获取表中的所有交易记录

```sql
SELECT * FROM trades;
```

## 破坏性查询

这些是会对数据库进行写入操作的查询。
通常这些查询不是必需的，因为 Freqtrade 会尝试自行处理所有数据库操作——或通过 API 或 Telegram 命令公开这些操作。

!!! Warning
    在运行以下任何查询之前，请确保您已备份数据库。

!!! Danger
    您也**绝对不应**在机器人连接到数据库时运行任何写入查询（`update`、`insert`、`delete`）。
    这可能导致并且很可能会导致数据损坏——最坏的情况是无法恢复。

### 修复在交易所手动平仓后交易仍显示为开启状态

!!! Warning
    在交易所手动卖出交易对不会被机器人检测到，机器人仍会尝试卖出。应尽可能使用 `/forceexit <tradeid>` 命令来实现相同操作。  
    强烈建议在进行任何手动修改前备份数据库文件。

!!! Note
    在使用 `/forceexit` 后通常无需此操作，因为强平订单会在下一次迭代时由机器人自动关闭。

```sql
UPDATE trades
SET is_open=0,
  close_date=<close_date>,
  close_rate=<close_rate>,
  close_profit = close_rate / open_rate - 1,
  close_profit_abs = (amount * <close_rate> * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason=<exit_reason>
WHERE id=<trade_ID_to_update>;
```

#### 示例

```sql
UPDATE trades
SET is_open=0,
  close_date='2020-06-20 03:08:45.103418',
  close_rate=0.19638016,
  close_profit=0.0496,
  close_profit_abs = (amount * 0.19638016 * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason='force_exit'  
WHERE id=31;
```

### 从数据库中移除交易

!!! Tip "使用 RPC 方法删除交易"
    考虑通过 Telegram 或 REST API 使用 `/delete <tradeid>` 命令。这是删除交易的推荐方式。

如果您仍希望直接从数据库中移除交易，可以使用以下查询。

!!! Danger
    某些系统（如 Ubuntu）在其 sqlite3 软件包中默认禁用外键。使用 sqlite 时，请确保在执行上述查询前运行 `PRAGMA foreign_keys = ON` 来启用外键。

```sql
DELETE FROM trades WHERE id = <tradeid>;

DELETE FROM trades WHERE id = 31;
```

!!! Warning
    这将从数据库中删除该交易记录。请确保您获取了正确的 id，并且**绝对不要**在没有 `where` 子句的情况下运行此查询。