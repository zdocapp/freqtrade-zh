# 订单流数据

本指南将引导您如何利用公共交易数据在 Freqtrade 中进行高级订单流分析。

!!! Warning "实验性功能"
    订单流功能目前处于测试阶段，未来版本中可能会有变更。请在 [Freqtrade GitHub 仓库](https://github.com/freqtrade/freqtrade/issues) 报告任何问题或反馈。
    该功能目前尚未与 freqAI 进行过测试，将这两个功能结合使用目前不在考虑范围内。

!!! Warning "性能提示"
    订单流需要原始交易数据。这些数据量相当大，可能导致初始启动缓慢，因为 freqtrade 需要下载最近 X 根 K 线的交易数据。此外，启用此功能会增加内存使用量。请确保拥有足够的可用资源。

## 快速开始

### 启用公共交易数据

在您的 `config.json` 文件中，将 `exchange` 部分下的 `use_public_trades` 选项设置为 true。

```json
"exchange": {
   ...
   "use_public_trades": true,
}
```

### 配置订单流处理

在 config.json 的 orderflow 部分定义您所需的订单流处理设置。在此处，您可以调整以下因素：

- `cache_size`: 缓存中保存多少个历史订单流蜡烛图数据，避免每次新蜡烛图时重新计算
- `max_candles`: 筛选获取交易数据的蜡烛图数量
- `scale`: 控制足迹图的价格分档大小
- `stacked_imbalance_range`: 定义需要考虑的最小连续失衡价格级别数量
- `imbalance_volume`: 过滤掉成交量低于此阈值的失衡情况
- `imbalance_ratio`: 过滤掉比率（卖盘与买盘成交量之差）低于此值的失衡情况

```json
"orderflow": {
    "cache_size": 1000, 
    "max_candles": 1500, 
    "scale": 0.5, 
    "stacked_imbalance_range": 3, //  needs at least this amount of imbalance next to each other
    "imbalance_volume": 1, //  filters out below
    "imbalance_ratio": 3 //  filters out ratio lower than
  },
```

## 下载回测用交易数据

要下载历史交易数据进行回测，请在 freqtrade download-data 命令中使用 --dl-trades 标志。

```bash
freqtrade download-data -p BTC/USDT:USDT --timerange 20230101- --trading-mode futures --timeframes 5m --dl-trades
```

!!! Warning "数据可用性"
    并非所有交易所都提供公开交易数据。对于受支持的交易所，如果您使用 `--dl-trades` 标志开始下载数据，但公开交易数据不可用，freqtrade 会发出警告。

## 访问订单流数据

激活后，您的数据框中将新增多个列：

``` python

dataframe["trades"] # Contains information about each individual trade.
dataframe["orderflow"] # Represents a footprint chart dict (see below)
dataframe["imbalances"] # Contains information about imbalances in the order flow.
dataframe["bid"] # Total bid volume 
dataframe["ask"] # Total ask volume
dataframe["delta"] # Difference between ask and bid volume.
dataframe["min_delta"] # Minimum delta within the candle
dataframe["max_delta"] # Maximum delta within the candle
dataframe["total_trades"] # Total number of trades
dataframe["stacked_imbalances_bid"] # List of price levels of stacked bid imbalance range beginnings
dataframe["stacked_imbalances_ask"] # List of price levels of stacked ask imbalance range beginnings
```

您可以在策略代码中访问这些列进行进一步分析。以下是一个示例：

``` python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # Calculating cumulative delta
    dataframe["cum_delta"] = cumulative_delta(dataframe["delta"])
    # Accessing total trades
    total_trades = dataframe["total_trades"]
    ...

def cumulative_delta(delta: Series):
    cumdelta = delta.cumsum()
    return cumdelta

```

### 足迹图 (`dataframe["orderflow"]`)

该列提供了不同价格水平上买卖订单的详细分解，为订单流动态提供了宝贵的洞察。配置中的 `scale` 参数决定了此表示的价格区间大小

`orderflow` 列包含一个具有以下结构的字典：

``` output
{
    "price": {
        "bid_amount": 0.0,
        "ask_amount": 0.0,
        "bid": 0,
        "ask": 0,
        "delta": 0.0,
        "total_volume": 0.0,
        "total_trades": 0
    }
}
```

#### 订单流列说明

- key: 价格区间 - 按 `scale` 间隔进行分箱
- `bid_amount`: 每个价格水平的总买入量
- `ask_amount`: 每个价格水平的总卖出量
- `bid`: 每个价格水平的买单数量
- `ask`: 每个价格水平的卖单数量
- `delta`: 每个价格水平上卖出量与买入量的差值
- `total_volume`: 每个价格水平的总成交量（卖出量 + 买入量）
- `total_trades`: 每个价格水平的总交易次数（卖单数 + 买单数）

通过利用这些特性，您可以基于订单流分析获得对市场情绪和潜在交易机会的宝贵洞察

### 原始交易数据 (`dataframe["trades"]`)

包含蜡烛图期间发生的单笔交易的列表。此数据可用于更精细地分析订单流动态

每个单独条目包含一个具有以下键的字典：

- `timestamp`: 交易时间戳。
- `date`: 交易日期。
- `price`: 交易价格。
- `amount`: 交易量。
- `side`: 买入或卖出。
- `id`: 交易唯一标识符。
- `cost`: 交易总成本（价格 * 数量）。

### 订单失衡 (`dataframe["imbalances"]`)

该列提供一个字典，包含订单流失衡信息。当特定价格水平的卖单量和买单量出现显著差异时，就会发生失衡现象。

每行数据格式如下——以价格为索引，对应的买卖失衡值作为列

``` output
{
    "price": {
        "bid_imbalance": False,
        "ask_imbalance": False
    }
}
```