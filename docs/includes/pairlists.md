## 配对列表与配对列表处理器

配对列表处理器用于定义交易机器人应交易的配对列表（pairlist）。它们通过配置设置中的 `pairlists` 部分进行配置。

在配置中，您可以使用静态配对列表（由 [`StaticPairList`](#static-pair-list) 配对列表处理器定义）和动态配对列表（由 [`VolumePairList`](#volume-pair-list) 和 [`PercentChangePairList`](#percent-change-pair-list) 配对列表处理器定义）。

此外，[`AgeFilter`](#agefilter)、[`DelistFilter`](#delistfilter)、[`PrecisionFilter`](#precisionfilter)、[`PriceFilter`](#pricefilter)、[`ShuffleFilter`](#shufflefilter)、[`SpreadFilter`](#spreadfilter) 和 [`VolatilityFilter`](#volatilityfilter) 作为配对列表过滤器，用于移除特定交易对和/或调整它们在配对列表中的位置。

如果使用多个配对列表处理器，它们将形成链式结构，所有配对列表处理器的组合结果构成了交易机器人用于交易和回测的最终配对列表。配对列表处理器按照配置顺序执行。您可以将 `StaticPairList`、`VolumePairList`、`ProducerPairList`、`RemotePairList`、`MarketCapPairList` 或 `PercentChangePairList` 定义为起始配对列表处理器。

非活跃市场总是会从最终配对列表中移除。明确列入黑名单的交易对（位于 `pair_blacklist` 配置设置中）也总是会从最终配对列表中移除。

### 交易对黑名单

交易对黑名单（通过配置文件中的 `exchange.pair_blacklist` 配置）禁止特定交易对进行交易。
最简单的例子是排除 `DOGE/BTC`——这将精确移除该交易对。

交易对黑名单也支持通配符（采用正则表达式风格）——因此 `BNB/.*` 将排除所有以 BNB 开头的交易对。
您也可以使用类似 `.*DOWN/BTC` 或 `.*UP/BTC` 的模式来排除杠杆代币（请检查您交易所的交易对命名规范！）

### 可用的交易对列表处理器

* [`StaticPairList`](#静态交易对列表)（默认选项，若未另行配置）
* [`VolumePairList`](#成交量交易对列表)
* [`PercentChangePairList`](#百分比变化交易对列表)
* [`ProducerPairList`](#producerpairlist)
* [`RemotePairList`](#remotepairlist)
* [`MarketCapPairList`](#市值交易对列表)
* [`AgeFilter`](#agefilter)
* [`DelistFilter`](#delistfilter)
* [`FullTradesFilter`](#fulltradesfilter)
* [`OffsetFilter`](#offsetfilter)
* [`PerformanceFilter`](#performancefilter)
* [`PrecisionFilter`](#precisionfilter)
* [`PriceFilter`](#pricefilter)
* [`ShuffleFilter`](#shufflefilter)
* [`SpreadFilter`](#spreadfilter)
* [`RangeStabilityFilter`](#rangestabilityfilter)
* [`VolatilityFilter`](#volatilityfilter)

!!! Tip "测试配对列表"
    配对列表配置可能相当棘手，难以正确设置。最好在 [网页服务器模式](freq-ui.md#webserver-mode) 下使用 freqUI，或使用 [`test-pairlist`](utils.md#test-pairlist) 实用子命令来快速测试您的配对列表配置。

#### 静态配对列表

默认情况下，使用 `StaticPairList` 方法，该方法使用配置中静态定义的配对白名单。该配对列表还支持通配符（采用正则表达式风格）——因此 `.*/BTC` 将包含所有以 BTC 作为计价货币的配对。

它使用 `exchange.pair_whitelist` 和 `exchange.pair_blacklist` 中的配置，在以下示例中，将交易 BTC/USDT 和 ETH/USDT——并阻止 BNB/USDT 交易。

`pair_*list` 参数均支持正则表达式——因此像 `.*/USDT` 这样的值将启用所有不在黑名单中的配对交易。

```json
"exchange": {
    "name": "...",
    // ... 
    "pair_whitelist": [
        "BTC/USDT",
        "ETH/USDT",
        // ...
    ],
    "pair_blacklist": [
        "BNB/USDT",
        // ...
    ]
},
"pairlists": [
    {"method": "StaticPairList"}
],
```

默认情况下，仅允许当前启用的配对。
要跳过对活跃市场的配对验证，请在 `StaticPairList` 配置中设置 `"allow_inactive": true`。
这对于回测已过期配对（如季度现货市场）非常有用。

当在"后续"位置（例如，在 VolumePairlist 之后）使用时，`'pair_whitelist'` 中的所有配对将被添加到配对列表的末尾。

#### 成交量配对列表

`VolumePairList` 采用按交易量对交易对进行排序/筛选的方法。它根据 `sort_key`（只能是 `quoteVolume`）选择前 `number_assets` 个顶级交易对。

当在交易对处理程序链中处于非首位（位于 StaticPairList 和其他交易对过滤器之后）时，`VolumePairList` 会考虑先前交易对处理程序的输出，并在此基础上增加按交易量进行的排序/选择。

当在交易对处理程序链中处于首位时，将忽略 `pair_whitelist` 配置设置。相反，`VolumePairList` 会从交易所所有可用市场中筛选出与计价货币匹配的顶级资产。

`refresh_period` 设置允许定义交易对列表的刷新周期（以秒为单位），默认为 1800 秒（30 分钟）。
`VolumePairList` 上的交易对列表缓存（`refresh_period`）仅适用于生成交易对列表。
筛选实例（非列表首位）不会应用任何缓存（在高级模式下仅缓存蜡烛图数据直至该周期结束），并始终使用最新数据。

默认情况下，`VolumePairList` 基于 ccxt 库报告的交易所行情数据：

* `quoteVolume` 是指过去 24 小时内计价（保证金）货币的交易量（买入或卖出）。

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "max_value": 8000000,
        "refresh_period": 1800
    }
],
```

您可以使用 `min_value` 定义最小交易量——这将过滤掉指定时间范围内交易量低于该值的交易对。
此外，您还可以使用 `max_value` 定义最大交易量——这将过滤掉指定时间范围内交易量高于该值的交易对。

##### 交易量交易对列表高级模式

`VolumePairList` 也可以在高级模式下运行，通过指定时间范围内的K线数据来构建交易量。它利用交易所的历史K线数据，构建典型价格（通过（开盘价+最高价+最低价）/3计算）并将典型价格与每根K线的交易量相乘。其总和即为给定时间范围内的 `quoteVolume`。这允许实现不同的场景：当使用较长周期和较大K线尺寸时，可获得更平滑的交易量；反之，当使用较短周期和较小K线时则效果相反。

为方便起见，可以指定 `lookback_days`，这将意味着使用1天K线进行回看。以下示例中的交易对列表将基于过去7天数据创建：

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "refresh_period": 86400,
        "lookback_days": 7
    }
],
```

!!! Warning "范围回看与刷新周期"
    当与 `lookback_days` 和 `lookback_timeframe` 结合使用时，`refresh_period` 不能小于以秒为单位的K线尺寸。否则将导致向交易所API发送不必要的请求。

!!! Warning "使用回望范围时的性能影响"
    如果在首位结合回望功能使用，基于范围的交易量计算可能会消耗大量时间和资源，因为它会下载所有可交易货币对的K线数据。因此强烈建议采用标准方法，先使用 `VolumeFilter` 缩小货币对列表范围，再进行后续的范围交易量计算。

??? Tip "不受支持的交易所"
    在某些交易所（如Gemini），常规的VolumePairList无法正常工作，因为其API未原生提供24小时交易量数据。可以通过使用K线数据构建交易量来解决此问题。
    要大致模拟24小时交易量，可采用以下配置方案。
    请注意，这些货币对列表每天仅刷新一次。

    ```json
    "pairlists": [
        {
            "method": "VolumePairList",
            "number_assets": 20,
            "sort_key": "quoteVolume",
            "min_value": 0,
            "refresh_period": 86400,
            "lookback_days": 1
        }
    ],
    ```

可通过更复杂的方法实现：使用 `lookback_timeframe` 设置K线周期，并通过 `lookback_period` 指定K线数量。以下示例将基于3天1小时K线的滚动周期来构建交易量货币对列表：

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "refresh_period": 3600,
        "lookback_timeframe": "1h",
        "lookback_period": 72
    }
],
```

!!! Note
    `VolumePairList` 不支持回测模式。

#### 百分比变化货币对列表

`PercentChangePairList` 根据货币对在过去24小时或高级选项中任意定义时间段内的价格百分比变化来筛选和排序货币对。这使得交易者能够重点关注经历显著价格波动（无论是正向还是负向）的资产。

**配置选项**

* `number_assets`: 指定基于24小时百分比变化选择的前几名交易对数量。
* `min_value`: 设置最小百分比变化阈值。百分比变化低于此值的交易对将被过滤掉。
* `max_value`: 设置最大百分比变化阈值。百分比变化高于此值的交易对将被过滤掉。
* `sort_direction`: 指定基于百分比变化对交易对进行排序的顺序。接受两个值：`asc`表示升序，`desc`表示降序。
* `refresh_period`: 定义配对列表刷新的时间间隔（以秒为单位）。默认为1800秒（30分钟）。
* `lookback_days`: 回溯天数。当选择`lookback_days`时，`lookback_timeframe`默认为1天。
* `lookback_timeframe`: 用于回溯周期的时间框架。
* `lookback_period`: 回溯的周期数。

当百分比变化配对列表在其他配对列表处理器之后使用时，它将基于这些处理器的输出进行操作。如果它是首位的配对列表处理器，它将从具有指定计价货币的所有可用市场中选择交易对。

`PercentChangePairList`使用通过ccxt库提供的交易所行情数据：
百分比变化计算为过去24小时内的价格变化。

!!! Note "Unsupported exchanges"
    在某些交易所（如 HTX）上，常规的 PercentChangePairList 无法正常工作，因为其 API 并未原生提供 24 小时价格变动百分比。这可以通过使用 K 线数据计算变动百分比来解决。要大致模拟 24 小时变动百分比，您可以使用以下配置。请注意，这些交易对列表每天仅刷新一次。
    ```json
    "pairlists": [
        {
            "method": "PercentChangePairList",
            "number_assets": 20,
            "min_value": 0,
            "refresh_period": 86400,
            "lookback_days": 1
        }
    ],
    ```

**从行情数据读取的示例配置**

```json
"pairlists": [
    {
        "method": "PercentChangePairList",
        "number_assets": 15,
        "min_value": -10,
        "max_value": 50
    }
],
```

在此配置中：

1. 根据过去 24 小时内最高价格变动百分比选择前 15 个交易对。
2. 仅考虑变动百分比在 -10% 到 50% 之间的交易对。

**从 K 线数据读取的示例配置**

```json
"pairlists": [
    {
        "method": "PercentChangePairList",
        "number_assets": 15,
        "sort_key": "percentage",
        "min_value": 0,
        "refresh_period": 3600,
        "lookback_timeframe": "1h",
        "lookback_period": 72
    }
],
```

此示例通过使用 `lookback_timeframe` 指定 K 线周期和 `lookback_period` 指定 K 线数量，基于 3 天 1 小时 K 线的滚动周期构建变动百分比交易对。

价格变动百分比使用以下公式计算，该公式表示当前 K 线收盘价与指定时间周期和回溯周期所定义的先前 K 线收盘价之间的百分比差异：

$$ 百分比变化 = (\frac{当前收盘价 - 前收盘价}{前收盘价}) * 100 $$

!!! Warning "回看范围与刷新周期"
    当与 `lookback_days` 和 `lookback_timeframe` 结合使用时，`refresh_period` 不能小于以秒为单位的蜡烛图尺寸。否则将导致向交易所 API 发出不必要的请求。

!!! Warning "使用回看范围时的性能影响"
    如果在首位结合回看功能使用，基于范围的百分比变化计算可能会消耗大量时间和资源，因为它会下载所有可交易对的蜡烛图数据。因此强烈建议先使用 `PercentChangePairList` 的标准方法来缩小交易对列表，再进行百分比变化计算。

!!! Note "回测"
    `PercentChangePairList` 不支持回测模式。

#### ProducerPairList

使用 `ProducerPairList`，您可以复用来自 [Producer](producer-consumer.md) 的交易对列表，而无需在每个消费者上显式定义交易对列表。

此交易对列表需要启用 [消费者模式](producer-consumer.md) 才能正常工作。

该交易对列表会针对当前交易所配置检查活跃交易对，以避免尝试在无效市场上进行交易。

您可以使用可选参数 `number_assets` 来限制配对列表的长度。使用 `"number_assets"=0` 或省略此键将导致复用当前设置下所有有效的生产者配对。

```json
"pairlists": [
    {
        "method": "ProducerPairList",
        "number_assets": 5,
        "producer_name": "default",
    }
],
```

!!! Tip "组合配对列表"
    此配对列表可与所有其他配对列表和过滤器结合使用，以进一步缩减配对列表，也可作为"附加"配对列表，叠加在已定义的配对上。
    `ProducerPairList` 也可按顺序多次使用，组合来自多个生产者的配对。
    显然在此类复杂配置中，生产者可能无法为所有配对提供数据，因此策略必须适应这种情况。

#### RemotePairList

它允许用户从远程服务器或 freqtrade 目录内本地存储的 json 文件获取配对列表，从而实现交易配对列表的动态更新和自定义。

RemotePairList 在配置设置的 pairlists 部分中定义。它使用以下配置选项：

```json
"pairlists": [
    {
        "method": "RemotePairList",
        "mode": "whitelist",
        "processing_mode": "filter",
        "pairlist_url": "https://example.com/pairlist",
        "number_assets": 10,
        "refresh_period": 1800,
        "keep_pairlist_on_failure": true,
        "read_timeout": 60,
        "bearer_token": "my-bearer-token",
        "save_to_file": "user_data/filename.json" 
    }
]
```

可选的 `mode` 选项指定配对列表应作为 `blacklist`（黑名单）还是 `whitelist`（白名单）使用。默认值为 "whitelist"。

RemotePairList 配置中的可选 `processing_mode` 选项决定了如何处​​理获取的配对列表。它可以有两个值："filter"（过滤）或 "append"（追加）。默认值为 "filter"。

在"filter"模式下，检索到的交易对列表将用作过滤器。只有同时存在于原始交易对列表和检索到的交易对列表中的交易对才会被包含在最终列表中，其他交易对将被过滤掉。

在"append"模式下，检索到的交易对列表会被添加到原始交易对列表中。两个列表中的所有交易对都将被包含在最终列表中，不进行任何过滤。

`pairlist_url`选项指定了远程服务器上交易对列表的URL地址，或者是本地文件的路径（如果前缀为file:///）。这允许用户使用远程服务器或本地文件作为交易对列表的数据源。

`save_to_file`选项在提供有效文件名时，会将处理后的交易对列表以JSON格式保存到该文件中。此选项为可选配置，默认情况下交易对列表不会保存到文件。

??? 示例 "多机器人共享交易对列表示例"

    `save_to_file` can be used to save the pairlist to a file with Bot1:

    ```json
    "pairlists": [
        {
            "method": "RemotePairList",
            "mode": "whitelist",
            "pairlist_url": "https://example.com/pairlist",
            "number_assets": 10,
            "refresh_period": 1800,
            "keep_pairlist_on_failure": true,
            "read_timeout": 60,
            "save_to_file": "user_data/filename.json" 
        }
    ]
    ```

    This saved pairlist file can be loaded by Bot2, or any additional bot with this configuration:

    ```json
    "pairlists": [
        {
            "method": "RemotePairList",
            "mode": "whitelist",
            "pairlist_url": "file:///user_data/filename.json",
            "number_assets": 10,
            "refresh_period": 10,
            "keep_pairlist_on_failure": true,
        }
    ]
    ```    

用户需要负责提供一个服务器或本地文件，该源应返回具有以下结构的JSON对象：

```json
{
    "pairs": ["XRP/USDT", "ETH/USDT", "LTC/USDT"],
    "refresh_period": 1800
}
```

`pairs`属性应包含一个字符串列表，列出机器人要使用的交易对。`refresh_period`属性为可选，指定交易对列表在刷新前应缓存的秒数。

可选的`keep_pairlist_on_failure`指定当远程服务器无法访问或返回错误时，是否应使用先前接收到的交易对列表。默认值为true。

可选的 `read_timeout` 参数指定了等待远程源响应的最长时间（以秒为单位），默认值为 60。

可选的 `bearer_token` 将被包含在请求的 Authorization 头部中。

!!! Note
    当服务器发生错误时，如果 `keep_pairlist_on_failure` 设置为 true，将保留最后收到的交易对列表；若设置为 false，则返回空交易对列表。

#### MarketCapPairList

`MarketCapPairList` 采用基于 CoinGecko 的市值排名对交易对进行排序/筛选。返回的交易对列表将根据其市值排名进行排序。

```json
"pairlists": [
    {
        "method": "MarketCapPairList",
        "number_assets": 20,
        "max_rank": 50,
        "refresh_period": 86400,
        "categories": ["layer-1"]
    }
]
```

`number_assets` 定义了交易对列表返回的最大交易对数量。`max_rank` 将决定创建/筛选交易对列表时使用的最大排名。需要注意的是，由于并非所有位于前 `max_rank` 市值的代币都会在您偏好的市场/计价货币/交易所组合中存在活跃交易对，因此部分代币可能不会出现在最终的交易对列表中。  
虽然支持使用大于 250 的 `max_rank` 值，但不建议这样做，因为这会导致向 CoinGecko 发起多次 API 调用，可能引发速率限制问题。

`refresh_period` 设置定义了市值排名数据刷新的时间间隔（以秒为单位）。默认值为 86,400 秒（1 天）。该配对列表缓存（`refresh_period`）同时适用于生成配对列表（当位于列表首位时）和过滤实例（当不位于列表首位时）。

`categories` 设置指定了从中选择代币的 [CoinGecko 类别](https://www.coingecko.com/en/categories)。默认值为空列表 `[]`，表示不应用类别过滤。
如果选择了错误的类别字符串，插件将打印 CoinGecko 提供的可用类别并执行失败。类别应使用其 ID，例如，对于 `https://www.coingecko.com/en/categories/layer-1`，类别 ID 为 `layer-1`。您可以传递多个类别，如 `["layer-1", "meme-token"]`，以从多个类别中进行选择。

对于像 1000PEPE/USDT 或 KPEPE/USDT:USDT 这样的代币，系统会尽最大努力进行检测，使用前缀 `1000` 和 `K` 来识别它们。

!!! Warning "类别过多"
    每添加一个类别，就意味着要向 CoinGecko 发起一次 API 调用。添加的类别越多，生成配对列表所需的时间就越长，并可能引发速率限制问题。

!!! Danger "CoinGecko 中的重复符号"
    CoinGecko 经常出现重复符号，即同一符号被用于不同的代币。Freqtrade 将按原样使用该符号并尝试在交易所中搜索。如果该符号存在，则会被使用。然而，Freqtrade 不会检查交易所上的符号是否是 CoinGecko 所指的 _预期_ 符号。这有时可能导致意外结果，尤其是在低交易量代币或 meme 币类别中。

#### AgeFilter

移除在交易所上市时间少于 `min_days_listed` 天（默认为 `10` 天）或多于 `max_days_listed` 天（默认为 `None`，表示无上限）的交易对。

当交易对首次在交易所上市时，在最初几天经历价格发现阶段，可能会遭遇巨大的价格下跌和波动。交易机器人常常会在交易对价格下跌结束前就买入而被套牢。

此过滤器允许 freqtrade 忽略那些上市时间未达到至少 `min_days_listed` 天或上市时间早于 `max_days_listed` 天的交易对。

#### DelistFilter

移除将在未来最多 `max_days_from_now` 天（默认为 `0`，表示无论多久后，移除所有未来将被下架的货币对）内从交易所下架的货币对。目前此过滤器仅支持以下交易所：

!!! Note "Available exchanges"
    退市过滤器仅在币安交易所可用，其中币安期货在模拟和实盘模式下均可工作，而币安现货限于实盘模式（出于技术原因）。

!!! Warning "Backtesting"
    `DelistFilter` 不支持回测模式。

#### FullTradesFilter

当交易仓位已满时（配置中 `max_open_trades` 未设置为 `-1` 时），将白名单缩减为仅包含交易中的货币对。

当交易仓位已满时，无需计算其余货币对（信息对除外）的指标，因为无法开立新交易。通过将白名单缩减为仅交易中的货币对，可以提高计算速度并减少 CPU 使用率。当交易仓位空闲时（交易关闭或配置中 `max_open_trades` 值增加），白名单将恢复正常状态。

当使用多个货币对列表过滤器时，建议将此过滤器置于主货币对列表正下方的第二个位置，这样当交易仓位已满时，机器人无需为其余过滤器下载数据。

!!! Warning "Backtesting"
    `FullTradesFilter` 不支持回测模式。

#### OffsetFilter

通过给定的 `offset` 值对传入的货币对列表进行偏移。

例如，它可以与 `VolumeFilter` 结合使用，以移除交易量排名前 X 的交易对。或者将较大的交易对列表拆分到两个机器人实例上运行。

以下示例展示了如何从交易对列表中移除前 10 个交易对，并获取接下来的 20 个（即取初始列表的第 10-30 项）：

```json
"pairlists": [
    // ...
    {
        "method": "OffsetFilter",
        "offset": 10,
        "number_assets": 20
    }
],
```

!!! Warning
    当 `OffsetFilter` 与 `VolumeFilter` 结合使用，将较大的交易对列表拆分到多个机器人时，
    由于 `VolumeFilter` 的刷新间隔可能存在细微差异，无法保证交易对不会出现重叠。

!!! Note
    如果偏移量大于传入交易对列表的总长度，将导致返回空的交易对列表。

#### PerformanceFilter

根据历史交易表现对交易对进行排序，排序规则如下：

1.  正收益的交易对。
2.  尚未有成交记录的交易对。
3.  负收益的交易对。

交易次数被用作平局决胜条件。

您可以使用 `minutes` 参数来仅考虑过去 X 分钟内的表现（滚动窗口）。
不定义此参数（或将其设置为 0）将使用全时段的表现数据。

可选的 `min_profit` 参数（比率为单位 -> 设置为 `0.01` 对应 1%）定义了交易对被纳入考虑所需的最低利润。
低于此水平的交易对将被过滤掉。
强烈不建议在不使用 `minutes` 参数的情况下单独使用此参数，因为这可能导致交易对列表为空且无法恢复。

```json
"pairlists": [
    // ...
    {
        "method": "PerformanceFilter",
        "minutes": 1440,  // rolling 24h
        "min_profit": 0.01  // minimal profit 1%
    }
],
```

由于此过滤器使用机器人过去的表现数据，因此会有一个启动期——并且仅当机器人在数据库中积累了几百笔交易后才应使用。

!!! Warning "回测"
    `PerformanceFilter` 不支持回测模式。

#### PrecisionFilter

过滤那些无法设置止损的低价值币种。

具体来说，如果止损价格因交易所精度舍入导致1%或以上的变动，即满足 `rounded(stop_price) <= rounded(stop_price * 0.99)` 时，交易对将被列入黑名单。其原理是避免交易价格极度接近最低交易边界的币种，这类币种无法设置合理的止损位。

!!! Tip "PrecisionFilter 在期货交易中无意义"
    上述情况不适用于空头交易。而对于多头交易，理论上仓位会先被强平。

!!! Warning "回测"
    `PrecisionFilter` 不支持使用多策略的回测模式。

#### PriceFilter

`PriceFilter` 支持按价格筛选交易对。目前支持以下价格过滤器：

* `min_price`
* `max_price`
* `max_value`
* `low_price_ratio`

`min_price` 设置会过滤掉价格低于指定值的交易对。若希望避免交易极低价格的币对时，此功能非常实用。
该选项默认禁用，仅当设置值 > 0 时生效。

`max_price` 设置会移除价格高于指定价格的交易对。如果您希望仅交易低价币种，此功能非常实用。
该选项默认禁用，仅当设置值 > 0 时生效。

`max_value` 设置会移除最小价值变动超过指定值的交易对。
当交易所存在不平衡的限制时，此功能非常有用。例如，当步长=1（即只能购买1、2、3枚代币，不能购买1.1枚）且代币价格因近期暴涨而处于高位（如20美元）时。
基于上述情况，您只能以20美元或40美元购买，而无法以25美元购买。
在从接收货币中扣除手续费的交易所（如币安），这可能导致高价值代币/金额因略低于限制而无法卖出。

`low_price_ratio` 设置会移除1个价格单位（点）涨幅超过 `low_price_ratio` 比率的交易对。
该选项默认禁用，仅当设置值 > 0 时生效。

对于 `PriceFilter`，必须至少应用其 `min_price`、`max_price` 或 `low_price_ratio` 设置中的一项。

计算示例：

SHITCOIN/BTC 的最低价格精度为 8 位小数。如果其价格为 0.00000011，那么下一个价格步长将是 0.00000012，比前一个价格高出约 9%。您可以通过使用 PriceFilter 并相应地将 `low_price_ratio` 设置为 0.09（9%）或将 `min_price` 设置为 0.00000011 来过滤掉该交易对。

!!! Warning "低价交易对"
    具有高"1 点波动"的低价交易对非常危险，因为它们通常流动性差，并且可能无法设置所需的止损单。这往往会导致巨大损失，因为价格需要四舍五入到下一个可交易价格——因此，您的止损可能不是预期的 -5%，而最终变成 -9%，这仅仅是价格取整的结果。

#### ShuffleFilter

对配对列表中的交易对进行随机排序。当您希望所有交易对具有相同优先级时，此过滤器可用于防止机器人更频繁地交易某些交易对。

默认情况下，ShuffleFilter 每根 K 线周期随机排序一次。
若要在每次迭代时都进行随机排序，请将 `"shuffle_frequency"` 设置为 `"iteration"`，而非默认的 `"candle"`。

``` json
    {
        "method": "ShuffleFilter", 
        "shuffle_frequency": "candle",
        "seed": 42
    }

```

!!! Tip
    您可以为该配对列表设置 `seed` 值以获得可重现的结果，这在重复回测会话时非常有用。如果未设置 `seed`，配对将以不可重复的随机顺序进行洗牌。ShuffleFilter 会自动检测运行模式，并仅在回测模式下应用 `seed`（如果设置了 `seed` 值）。

#### SpreadFilter

移除买卖差价高于指定比率 `max_spread_ratio`（默认为 `0.005`）的交易对。

示例：

如果 `DOGE/BTC` 的最高买价为 0.00000026，最低卖价为 0.00000027，则比率计算为：`1 - 买价/卖价 ≈ 0.037`，该值 `> 0.005`，因此该交易对将被过滤掉。

#### RangeStabilityFilter

移除在过去 `lookback_days` 天内最低价与最高价之差低于 `min_rate_of_change` 或高于 `max_rate_of_change` 的交易对。由于此过滤器需要额外数据，结果会缓存 `refresh_period` 时长。

以下示例中：
如果过去 10 天的交易波动范围 <1% 或 >99%，则将该交易对从白名单中移除。

```json
"pairlists": [
    {
        "method": "RangeStabilityFilter",
        "lookback_days": 10,
        "min_rate_of_change": 0.01,
        "max_rate_of_change": 0.99,
        "refresh_period": 86400
    }
]
```

添加 `"sort_direction": "asc"` 或 `"sort_direction": "desc"` 可启用此配对列表的排序功能。

!!! Tip
    该过滤器可用于自动移除交易范围极小的稳定币对，这类币对极难通过交易获利。
    此外，它还可用于自动移除在给定时间段内波动率极高或极低的交易对。

#### VolatilityFilter

波动率是交易对历史价格随时间变化的程度，通过对数日收益的标准差来衡量。虽然实际分布可能有所不同，但收益通常被假设为正态分布。在正态分布中，68%的观测值落在一个标准差范围内，95%的观测值落在两个标准差范围内。假设波动率为0.05，意味着30天中有20天的预期收益将低于5%（一个标准差）。波动率是预期收益偏差的正比率，可能大于1.00。请参考维基百科对[`波动率`](https://en.wikipedia.org/wiki/Volatility_(finance))的定义。

该过滤器会在`lookback_days`天内的平均波动率低于`min_volatility`或高于`max_volatility`时移除交易对。由于这是需要额外数据的过滤器，结果会缓存`refresh_period`时间。

该过滤器可用于将交易对筛选至特定波动率范围，或避免波动率极高的交易对。

在以下示例中：
如果过去10天的波动率不在0.05-0.50范围内，则将该交易对从白名单中移除。该过滤器每24小时应用一次。

```json
"pairlists": [
    {
        "method": "VolatilityFilter",
        "lookback_days": 10,
        "min_volatility": 0.05,
        "max_volatility": 0.50,
        "refresh_period": 86400
    }
]
```

添加 `"sort_direction": "asc"` 或 `"sort_direction": "desc"` 可启用该配对列表的排序模式。

### 配对列表处理器的完整示例

以下示例将 `BNB/BTC` 加入黑名单，使用包含 `20` 个资产的 `VolumePairList`，按 `quoteVolume` 对交易对进行排序，然后使用 [`DelistFilter`](#delistfilter) 和 [`AgeFilter`](#agefilter) 过滤即将下架的交易对，并移除上市时间少于10天的交易对。随后应用 [`PrecisionFilter`](#precisionfilter) 和 [`PriceFilter`](#pricefilter)，过滤掉1个价格单位变动超过1%的所有资产。接着应用 [`SpreadFilter`](#spreadfilter) 和 [`VolatilityFilter`](#volatilityfilter)，最后使用预设的随机种子对交易对进行随机打乱。

```json
"exchange": {
    "pair_whitelist": [],
    "pair_blacklist": ["BNB/BTC"]
},
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume"
    },
    {
        "method": "DelistFilter",
        "max_days_from_now": 0,
    },
    {"method": "AgeFilter", "min_days_listed": 10},
    {"method": "PrecisionFilter"},
    {"method": "PriceFilter", "low_price_ratio": 0.01},
    {"method": "SpreadFilter", "max_spread_ratio": 0.005},
    {
        "method": "RangeStabilityFilter",
        "lookback_days": 10,
        "min_rate_of_change": 0.01,
        "refresh_period": 86400
    },
    {
        "method": "VolatilityFilter",
        "lookback_days": 10,
        "min_volatility": 0.05,
        "max_volatility": 0.50,
        "refresh_period": 86400
    },
    {"method": "ShuffleFilter", "seed": 42}
],
```