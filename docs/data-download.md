# 数据下载

## 获取回测和超参数优化所需数据

要下载回测和超参数优化所需的K线数据（OHLCV），请使用 `freqtrade download-data` 命令。

如果未指定额外参数，freqtrade将下载最近30天的 `"1m"` 和 `"5m"` 时间框架数据。
交易所和交易对将从 `config.json` 中获取（如果使用 `-c/--config` 指定）。
未提供配置文件时，`--exchange` 参数变为必需。

您可以使用相对时间范围（`--days 20`）或绝对起始点（`--timerange 20200101-`）。对于增量下载，建议使用相对时间范围方法。

!!! Tip "提示：更新现有数据"
    如果您的数据目录中已有回测数据，并希望将这些数据更新至今天，freqtrade将自动计算现有交易对缺失的时间范围，并从最新可用时间点开始下载直到"当前时间"，此时无需使用 `--days` 或 `--timerange` 参数。Freqtrade将保留现有数据，仅下载缺失数据。  
    如果您在添加新交易对后更新现有数据（新交易对尚无数据），请使用 `--new-pairs-days xx` 参数。新交易对将下载指定天数的数据，而旧交易对仅更新缺失数据。

### 使用方法

--8<-- "commands/download-data.md"

!!! Tip "下载单一计价货币的全部数据"
    通常，您可能需要下载特定计价货币的所有交易对数据。此时，可以使用以下简写方式：
    `freqtrade download-data --exchange binance --pairs ".*/USDT" <...>`。所提供的"pairs"字符串将扩展为包含该交易所所有活跃交易对。
    若需同时下载非活跃（已下架）交易对的数据，请在命令中添加 `--include-inactive-pairs` 参数。

!!! Note "启动周期"
    `download-data` 是一个独立于策略的命令。其理念是首次下载大量数据后，再逐步增加存储的数据量。

    For that reason, `download-data` does not care about the "startup-period" defined in a strategy. It's up to the user to download additional days if the backtest should start at a specific point in time (while respecting startup period).

### 开始下载

一个非常简单的命令（假设存在可用的 `config.json` 文件）示例如下。

```bash
freqtrade download-data --exchange binance
```

这将下载配置文件中定义的所有货币对的历史K线（OHLCV）数据。

或者，直接指定交易对

```bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT
```

或使用正则表达式（本例中用于下载所有活跃的USDT交易对）

```bash
freqtrade download-data --exchange binance --pairs ".*/USDT"
```

### 其他说明

* 若要使用不同于交易所特定默认目录的目录，请使用 `--datadir user_data/data/some_directory`。
* 若要更改用于下载历史数据的交易所，可使用 `--exchange <交易所名称>` - 或指定不同的配置文件。
* 若要使用其他目录中的 `pairs.json` 文件，请使用 `--pairs-file 其他目录/pairs.json`。
* 若仅下载10天的历史K线（OHLCV）数据，请使用 `--days 10`（默认为30天）。
* 若要从固定起始点下载历史K线（OHLCV）数据，请使用 `--timerange 20200101-` - 这将下载从2020年1月1日起的所有数据。
* 如果数据已存在，则给定的起始点将被忽略，仅下载截至今日的缺失数据。
* 使用 `--timeframes` 指定要下载的历史K线（OHLCV）数据的时间周期。默认为 `--timeframes 1m 5m`，将下载1分钟和5分钟数据。
* 若要使用配置文件中定义的交易所、时间周期和交易对列表，请使用 `-c/--config` 选项。使用此选项时，脚本将使用配置中定义的白名单作为要下载数据的货币对列表，且不需要 pairs.json 文件。您可以将 `-c/--config` 与大多数其他选项结合使用。

??? Note "权限拒绝错误"
    如果您的配置目录 `user_data` 是由 docker 创建的，可能会遇到以下错误：

    ```
    cp: cannot create regular file 'user_data/data/binance/pairs.json': Permission denied
    ```

    You can fix the permissions of your user-data directory as follows:

    ```
    sudo chown -R $UID:$GID user_data
    ```

### 在当前时间范围前下载额外数据

假设您已下载了2022年以来的所有数据（`--timerange 20220101-`），但现在希望使用更早的数据进行回测。
您可以通过结合使用 `--prepend` 标志和指定结束日期的 `--timerange` 来实现。

``` bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT --prepend --timerange 20210101-20220101
```

!!! Note
    在此模式下，如果数据已存在，Freqtrade 将忽略结束日期，并将结束日期更新为现有数据的起始点。

### 数据格式

Freqtrade 目前支持以下数据格式：

* `feather` - 基于 Apache Arrow 的数据格式
* `json` - 纯文本 json 文件
* `jsongz` - gzip 压缩的 json 文件版本
* `parquet` - 列式数据存储（仅限 OHLCV 数据）

默认情况下，OHLCV 数据和交易数据均以 `feather` 格式存储。

可通过分别使用 `--data-format-ohlcv` 和 `--data-format-trades` 命令行参数来更改此设置。
为使此更改持久化，您还应在配置中添加以下代码片段，以避免每次都需要输入上述参数：

``` jsonc
    // ...
    "dataformat_ohlcv": "feather",
    "dataformat_trades": "feather",
    // ...
```

如果在下载过程中更改了默认数据格式，则配置文件中的 `dataformat_ohlcv` 和 `dataformat_trades` 键也需要调整为所选数据格式。

!!! Note
    您可以使用 [convert-data](#sub-command-convert-data) 和 [convert-trade-data](#sub-command-convert-trade-data) 方法在不同数据格式之间进行转换。

#### 数据格式比较

以下比较基于以下数据，并使用 Linux 的 `time` 命令进行。

```
Found 6 pair / timeframe combinations.
+----------+-------------+--------+---------------------+---------------------+
|     Pair |   Timeframe |   Type |                From |                  To |
|----------+-------------+--------+---------------------+---------------------|
| BTC/USDT |          5m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:25:00 |
| ETH/USDT |          1m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:26:00 |
| BTC/USDT |          1m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:30:00 |
| XRP/USDT |          5m |   spot | 2018-05-04 08:10:00 | 2022-09-13 19:15:00 |
| XRP/USDT |          1m |   spot | 2018-05-04 08:11:00 | 2022-09-13 19:22:00 |
| ETH/USDT |          5m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:20:00 |
+----------+-------------+--------+---------------------+---------------------+
```

计时采用不太科学的方式，通过以下命令强制将数据读入内存。

``` bash
time freqtrade list-data --show-timerange --data-format-ohlcv <dataformat>
```

|  格式 | 大小 | 耗时 |
|------------|-------------|-------------|
| `feather` | 72Mb | 3.5s |
| `json` | 149Mb | 25.6s |
| `jsongz` | 39Mb | 27s |
| `parquet` | 83Mb | 3.8s |

大小数据取自上述时间范围内的 BTC/USDT 1分钟现货组合。

为获得最佳性能与大小的平衡，我们推荐使用默认的 feather 格式或 parquet 格式。

### 交易对文件

除了 `config.json` 中的白名单外，还可以使用 `pairs.json` 文件。
例如，如果您使用币安：

* 创建目录 `user_data/data/binance`，并在该目录中复制或创建 `pairs.json` 文件。
* 更新 `pairs.json` 文件，包含您感兴趣的货币对。

```bash
mkdir -p user_data/data/binance
touch user_data/data/binance/pairs.json
```

`pairs.json` 文件的格式为简单的 JSON 列表。
此文件允许混合不同的计价货币，因为它仅用于数据下载。

``` json
[
    "ETH/BTC",
    "ETH/USDT",
    "BTC/USDT",
    "XRP/ETH"
]
```

!!! Note
    仅当未加载配置（通过隐式命名或通过 `--config` 标志）时，才会使用 `pairs.json` 文件。
    您可以通过 `--pairs-file pairs.json` 强制使用此文件——但我们建议在配置中使用配对列表，无论是通过 `exchange.pair_whitelist` 还是配置中的 `pairs` 设置。

## 子命令 convert data

--8<-- "commands/convert-data.md"

### 数据转换示例

以下命令将把 `~/.freqtrade/data/binance` 中所有可用的蜡烛图（OHLCV）数据从 json 格式转换为 jsongz 格式，在此过程中节省磁盘空间。
它还将删除原始的 json 数据文件（`--erase` 参数）。

``` bash
freqtrade convert-data --format-from json --format-to jsongz --datadir ~/.freqtrade/data/binance -t 5m 15m --erase
```

## 子命令 convert trade data

--8<-- "commands/convert-trade-data.md"

### 交易数据转换示例

以下命令将把 `~/.freqtrade/data/kraken` 中所有可用的交易数据从 jsongz 格式转换为 json 格式。
它还将删除原始的 jsongz 数据文件（`--erase` 参数）。

``` bash
freqtrade convert-trade-data --format-from jsongz --format-to json --datadir ~/.freqtrade/data/kraken --erase
```

## 子命令 trades to ohlcv

当您需要使用 `--dl-trades`（仅限 kraken）下载数据时，将交易数据转换为 ohlcv 数据是最后一步。
此命令允许您为额外的时间帧重复此最后一步，而无需重新下载数据。

--8<-- "commands/trades-to-ohlcv.md"

### 交易数据转 OHLCV 示例

``` bash
freqtrade trades-to-ohlcv --exchange kraken -t 5m 1h 1d --pairs BTC/EUR ETH/EUR
```

## 子命令 list-data

您可以使用 `list-data` 子命令获取已下载数据的列表。

--8<-- "commands/list-data.md"

### 示例 list-data

```bash
> freqtrade list-data --userdir ~/.freqtrade/user_data/

              Found 33 pair / timeframe combinations.
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━┓
┃          Pair ┃                                 Timeframe ┃ Type ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━┩
│       ADA/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│       ADA/ETH │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│       ETH/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│      ETH/USDT │                  5m, 15m, 30m, 1h, 2h, 4h │ spot │
└───────────────┴───────────────────────────────────────────┴──────┘

```

显示所有交易数据，包括起止时间范围

``` bash
> freqtrade list-data --show --trades
                     Found trades data for 1 pair.                     
┏━━━━━━━━━┳━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━┓
┃    Pair ┃ Type ┃                From ┃                  To ┃ Trades ┃
┡━━━━━━━━━╇━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━┩
│ XRP/ETH │ spot │ 2019-10-11 00:00:11 │ 2019-10-13 11:19:28 │  12477 │
└─────────┴──────┴─────────────────────┴─────────────────────┴────────┘

```

## 交易（tick）数据

默认情况下，`download-data` 子命令下载的是蜡烛图（OHLCV）数据。大多数交易所也通过其 API 提供历史交易数据。
如果您需要多种不同的时间框架，这些数据会很有用，因为它只需下载一次，然后在本地重采样到所需的时间框架。

由于此数据默认情况下体积较大，文件默认使用 feather 文件格式。它们存储在您的数据目录中，命名约定为 `<pair>-trades.feather`（例如 `ETH_BTC-trades.feather`）。与历史 OHLCV 数据一样，也支持增量模式，因此每周使用 `--days 8` 下载一次数据将创建一个增量数据存储库。

要使用此模式，只需在调用中添加 `--dl-trades`。这将切换下载方法以下载交易数据。
如果同时提供了 `--convert`，重采样步骤将自动进行，并覆盖给定货币对/时间框架组合可能已存在的 OHLCV 数据。

!!! Warning "Do not use"
    除非您是 Kraken 用户（Kraken 不提供历史 OHLCV 数据），否则不应使用此方法。  
    大多数其他交易所都提供具有足够历史记录的 OHLCV 数据，因此通过该方法下载多个时间框架的数据仍然比下载交易数据快得多。

!!! Note "Kraken user"
    Kraken 用户在开始下载数据前应阅读[此说明](exchanges.md#historic-kraken-data)。

调用示例：

```bash
freqtrade download-data --exchange kraken --pairs XRP/EUR ETH/EUR --days 20 --dl-trades
```

!!! Note
    虽然此方法使用异步调用，但由于需要前一次调用的结果来生成对交易所的下一个请求，因此速度会很慢。

## 下一步

很好，您现在已下载了一些数据，可以开始[回测](backtesting.md)您的策略了。