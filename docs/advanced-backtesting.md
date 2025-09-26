# 高级回测分析

## 分析买入/入场和卖出/离场标签

了解策略如何根据用于标记不同买入条件的买入/入场标签来执行是很有帮助的。您可能希望查看比默认回测输出提供的更复杂的每个买入和卖出条件的统计数据。您可能还希望确定导致交易开仓的信号蜡烛上的指标值。

!!! Note
    以下买入原因分析仅适用于回测，*不适用于超参数优化*。

我们需要将 `--export` 选项设置为 `signals` 来运行回测，以启用信号**和**交易的导出：

``` bash
freqtrade backtesting -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange=<timerange> --export=signals
```

这将告诉 freqtrade 输出一个包含策略、交易对以及导致入场和出场信号的蜡烛图 DataFrame 的 pickle 字典。
根据您的策略产生的入场信号数量，此文件可能会变得相当大，因此请定期检查您的 `user_data/backtest_results` 文件夹以删除旧的导出文件。

在运行下一次回测之前，请确保删除旧的回测结果，或使用 `--cache none` 选项运行回测，以确保不使用任何缓存的结果。

如果一切顺利，您现在应该在 `user_data/backtest_results` 文件夹中看到 `backtest-result-{timestamp}_signals.pkl` 和 `backtest-result-{timestamp}_exited.pkl` 文件。

要分析入场/出场标签，我们现在需要使用 `freqtrade backtesting-analysis` 命令
并配合 `--analysis-groups` 选项，该选项接受以空格分隔的参数：

``` bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 1 2 3 4 5
```

此命令将读取最近的回测结果。`--analysis-groups` 选项用于
指定各种表格输出，显示每个组别或交易的利润，
范围从最简单的 (0) 到最详细的按交易对、按买入和按卖出标签 (4)：

* 0: 按 enter_tag 统计的整体胜率和利润摘要
* 1: 按 enter_tag 分组的利润摘要
* 2: 按 enter_tag 和 exit_tag 分组的利润摘要
* 3: 按交易对和 enter_tag 分组的利润摘要
* 4: 按交易对、enter_tag 和 exit_tag 分组的利润摘要（这可能会变得相当庞大）
* 5: 按 exit_tag 分组的利润摘要

通过运行 `-h` 选项可以查看更多可用选项。

### 使用 backtest-filename

默认情况下，`backtesting-analysis` 会处理 `user_data/backtest_results` 目录中最近的回测结果。
如果你想分析较早的回测结果，可以使用 `--backtest-filename` 选项来指定所需的文件。这使你可以随时重新访问和分析历史回测输出，只需提供相关回测结果的文件名：

``` bash
freqtrade backtesting-analysis -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange <timerange> --export signals --backtest-filename backtest-result-2025-03-05_20-38-34.zip
```

你应该会在日志中看到类似以下的输出，其中包含导出的带时间戳的文件名：

```
2022-06-14 16:28:32,698 - freqtrade.misc - INFO - dumping json to "mystrat_backtest-2022-06-14_16-28-32.json"
```

然后你可以在 `backtesting-analysis` 中使用该文件名：

```
freqtrade backtesting-analysis -c <config.json> --backtest-filename=mystrat_backtest-2022-06-14_16-28-32.json
```

要使用不同结果目录中的回测结果，可通过 `--backtest-directory` 指定目标目录

``` bash
freqtrade backtesting-analysis -c <config.json> --backtest-directory custom_results/ --backtest-filename mystrat_backtest-2022-06-14_16-28-32.json
```

### 调整显示的买入标签和卖出标签

若只需在输出结果中显示特定的买入和卖出标签，可使用以下两个选项：

```
--enter-reason-list : Space-separated list of enter signals to analyse. Default: "all"
--exit-reason-list : Space-separated list of exit signals to analyse. Default: "all"
```

例如：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss
```

### 输出信号蜡烛指标

`freqtrade backtesting-analysis` 的真正优势在于能够输出信号蜡烛上的指标数值，便于对买入信号指标进行细粒度检查和调优。如需为指定指标组打印单独列，请使用 `--indicator-list` 选项：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss --indicator-list rsi rsi_1h bb_lowerband ema_9 macd macdsignal
```

相关指标必须存在于策略的主数据框中（主时间框架或信息时间框架均可），否则该脚本输出中将直接忽略这些指标。

!!! Note "指标列表"
    指标数值将同时显示在入场点和出场点。若指定 `--indicator-list all`，则仅显示入场点指标，以避免因策略差异导致列表过长。

分析结果中已包含一系列与蜡烛线和交易相关的字段，这些字段可通过添加到指标列表自动调用，具体包括：

- **open_date     :** 交易开仓时间
- **close_date    :** 交易平仓时间
- **min_rate      :** 持仓期间最低价格
- **max_rate      :** 持仓期间最高价格
- **open          :** 信号K线开盘价
- **close         :** 信号K线收盘价
- **high          :** 信号K线最高价
- **low           :** 信号K线最低价
- **volume        :** 信号K线成交量
- **profit_ratio  :** 交易收益率
- **profit_abs    :** 交易绝对收益

#### 指标值示例输出

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen 
```

在本示例中，
我们旨在展示交易入场点和出场点的 `chikou_span` 和 `tenkan_sen` 指标值。

指标值的示例输出可能如下所示：

| 交易对      | 开仓时间                 | 入场原因 | 离场原因 | 入场时 chikou_span | 入场时 tenkan_sen | 离场时 chikou_span | 离场时 tenkan_sen |
|-----------|---------------------------|--------------|-------------|---------------------|--------------------|--------------------|-------------------|
| DOGE/USDT | 2024-07-06 00:35:00+00:00 |              | exit_signal | 0.105               | 0.106              | 0.105              | 0.107             |
| BTC/USDT  | 2024-08-05 14:20:00+00:00 |              | roi         | 54643.440           | 51696.400          | 54386.000          | 52072.010         |

如表所示，`chikou_span (entry)` 代表交易入场时的指标值，而 `chikou_span (exit)` 则反映其离场时的数值。这种对指标值的详细展示增强了分析能力。

指标名称添加的 `(entry)` 和 `(exit)` 后缀用于区分交易入场点和离场点的数值。

!!! Note "全局交易指标"
    某些全局交易指标不带有 `(entry)` 或 `(exit)` 后缀。这些指标包括：`pair`, `stake_amount`, `max_stake_amount`, `amount`, `open_date`, `close_date`, `open_rate`, `close_rate`, `fee_open`, `fee_close`, `trade_duration`, `profit_ratio`, `profit_abs`, `exit_reason`, `initial_stop_loss_abs`, `initial_stop_loss_ratio`, `stop_loss_abs`, `stop_loss_ratio`, `min_rate`, `max_rate`, `is_open`, `enter_tag`, `leverage`, `is_short`, `open_timestamp`, `close_timestamp` 以及 `orders`

#### 基于入场或离场信号筛选指标

`--indicator-list` 选项默认同时显示入场和离场信号的指标值。若需仅筛选入场信号的指标值，可使用 `--entry-only` 参数。同理，要仅显示离场信号的指标值，则使用 `--exit-only` 参数。

示例：显示入场信号的指标值：

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --entry-only
```

示例：显示离场信号的指标值：

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --exit-only
```

!!! note 
    使用这些过滤器时，指标名称将不会带有 `(entry)` 或 `(exit)` 后缀。

### 按日期筛选交易结果

要仅显示回测时间范围内特定日期之间的交易，请使用标准的 `timerange` 选项，格式为 `YYYYMMDD-[YYYYMMDD]`：

```
--timerange : Timerange to filter output trades, start date inclusive, end date exclusive. e.g. 20220101-20221231
```

例如，如果您的回测时间范围是 `20220101-20221231`，但只想输出 1 月份的交易：

```bash
freqtrade backtesting-analysis -c <config.json> --timerange 20220101-20220201
```

### 打印被拒绝的信号

使用 `--rejected-signals` 选项可打印被拒绝的信号。

```bash
freqtrade backtesting-analysis -c <config.json> --rejected-signals
```

### 将表格写入 CSV 文件

某些表格输出可能变得很大，因此将其打印到终端并不理想。
使用 `--analysis-to-csv` 选项可禁用向标准输出打印表格，并将其写入 CSV 文件。

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv
```

默认情况下，这将为你在 `backtesting-analysis` 命令中指定的每个输出表格写入一个文件，例如：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --rejected-signals --analysis-groups 0 1
```

这将写入 `user_data/backtest_results` 目录：

* rejected_signals.csv
* group_0.csv
* group_1.csv

要覆盖文件写入位置，还需指定 `--analysis-csv-path` 选项。

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --analysis-csv-path another/data/path/
```