# 生产者 / 消费者模式

freqtrade 提供了一种机制，允许一个实例（也称为 `consumer`）通过消息 WebSocket 监听来自上游 freqtrade 实例（也称为 `producer`）的消息。主要包括 `analyzed_df` 和 `whitelist` 消息。这使得可以在多个机器人中重复使用交易对的已计算指标（和信号），而无需多次计算它们。

有关设置消息 WebSocket 的 `api_server` 配置（这将作为您的生产者），请参阅 Rest API 文档中的 [消息 WebSocket](rest-api.md#message-websocket)。

!!! Note
    我们强烈建议将 `ws_token` 设置为随机且仅您自己知晓的值，以避免未经授权访问您的机器人。

## 配置

通过在消费者的配置文件中添加 `external_message_consumer` 部分来启用对实例的订阅。

```json
{
    //...
   "external_message_consumer": {
        "enabled": true,
        "producers": [
            {
                "name": "default", // This can be any name you'd like, default is "default"
                "host": "127.0.0.1", // The host from your producer's api_server config
                "port": 8080, // The port from your producer's api_server config
                "secure": false, // Use a secure websockets connection, default false
                "ws_token": "sercet_Ws_t0ken" // The ws_token from your producer's api_server config
            }
        ],
        // The following configurations are optional, and usually not required
        // "wait_timeout": 300,
        // "ping_timeout": 10,
        // "sleep_time": 10,
        // "remove_entry_exit_signals": false,
        // "message_size_limit": 8
    }
    //...
}
```

| 参数 | 描述 |
|------------|-------------|
| `enabled` | **必需。** 启用消费者模式。如果设为 false，则忽略本节所有其他设置。<br>*默认值：`false`。<br>**数据类型：** 布尔值。|
| `producers` | **必需。** 生产者列表<br>**数据类型：** 数组。|
| `producers.name` | **必需。** 此生产者的名称。如果使用多个生产者，则必须在调用 `get_producer_pairs()` 和 `get_producer_df()` 时使用此名称。<br>**数据类型：** 字符串|
| `producers.host` | **必需。** 生产者的主机名或 IP 地址。<br>**数据类型：** 字符串|
| `producers.port` | **必需。** 与上述主机匹配的端口。<br>*默认值：`8080`。<br>**数据类型：** 整数|
| `producers.secure` | **可选。** 在 WebSocket 连接中使用 SSL。默认 False。<br>**数据类型：** 字符串|
| `producers.ws_token` | **必需。** 生产者上配置的 `ws_token`。<br>**数据类型：** 字符串|
| | **可选设置**|
| `wait_timeout` | 未收到消息时再次 ping 的超时时间。<br>*默认值：`300`。<br>**数据类型：** 整数 - 单位秒。|
| `ping_timeout` | Ping 超时时间<br>*默认值：`10`。<br>**数据类型：** 整数 - 单位秒。|
| `sleep_time` | 重试连接前的休眠时间。<br>*默认值：`10`。<br>**数据类型：** 整数 - 单位秒。|
| `remove_entry_exit_signals` | 在接收数据帧时从数据帧中移除信号列（将其设为 0）。<br>*默认值：`false`。<br>**数据类型：** 布尔值。|
| `initial_candle_limit` | 预期从生产者获取的初始 K 线数量。<br>*默认值：`1500`。<br>**数据类型：** 整数 - K 线数量。|
| `message_size_limit` | 每条消息的大小限制<br>*默认值：`8`。<br>**数据类型：** 整数 - 单位兆字节。|

除了（或同时）在 `populate_indicators()` 中计算指标外，跟随者实例会监听与生产者实例消息（或高级配置中的多个生产者实例）的连接，并请求生产者针对活跃白名单中每个交易对最近分析过的数据帧。

这样，消费者实例将获得已分析数据帧的完整副本，无需自行计算。

## 使用示例

### 示例 - 生产者策略

一个包含多个指标的简单策略。策略本身无需特殊考量。

```py
class ProducerStrategy(IStrategy):
    #...
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Calculate indicators in the standard freqtrade way which can then be broadcast to other instances
        """
        dataframe['rsi'] = ta.RSI(dataframe)
        bollinger = qtpylib.bollinger_bands(qtpylib.typical_price(dataframe), window=20, stds=2)
        dataframe['bb_lowerband'] = bollinger['lower']
        dataframe['bb_middleband'] = bollinger['mid']
        dataframe['bb_upperband'] = bollinger['upper']
        dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Populates the entry signal for the given dataframe
        """
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], self.buy_rsi.value)) &
                (dataframe['tema'] <= dataframe['bb_middleband']) &
                (dataframe['tema'] > dataframe['tema'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

!!! Tip "FreqAI"
    你可以利用此功能在性能强大的机器上设置 [FreqAI](freqai.md)，同时在树莓派等简易设备上运行消费者实例，这些设备能以不同方式解析生产者生成的信号。

### 示例 - 消费者策略

这是一个逻辑等效的策略，其本身不计算任何指标，但将拥有相同的已分析数据帧，可基于生产者在生产者端计算的指标做出交易决策。本例中消费者采用相同的入场条件，但这并非必需。消费者可使用不同的逻辑进行入场/出场交易，仅使用指定的指标。

```py
class ConsumerStrategy(IStrategy):
    #...
    process_only_new_candles = False # required for consumers

    _columns_to_expect = ['rsi_default', 'tema_default', 'bb_middleband_default']

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Use the websocket api to get pre-populated indicators from another freqtrade instance.
        Use `self.dp.get_producer_df(pair)` to get the dataframe
        """
        pair = metadata['pair']
        timeframe = self.timeframe

        producer_pairs = self.dp.get_producer_pairs()
        # You can specify which producer to get pairs from via:
        # self.dp.get_producer_pairs("my_other_producer")

        # This func returns the analyzed dataframe, and when it was analyzed
        producer_dataframe, _ = self.dp.get_producer_df(pair)
        # You can get other data if the producer makes it available:
        # self.dp.get_producer_df(
        #   pair,
        #   timeframe="1h",
        #   candle_type=CandleType.SPOT,
        #   producer_name="my_other_producer"
        # )

        if not producer_dataframe.empty:
            # If you plan on passing the producer's entry/exit signal directly,
            # specify ffill=False or it will have unintended results
            merged_dataframe = merge_informative_pair(dataframe, producer_dataframe,
                                                      timeframe, timeframe,
                                                      append_timeframe=False,
                                                      suffix="default")
            return merged_dataframe
        else:
            dataframe[self._columns_to_expect] = 0

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Populates the entry signal for the given dataframe
        """
        # Use the dataframe columns as if we calculated them ourselves
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi_default'], self.buy_rsi.value)) &
                (dataframe['tema_default'] <= dataframe['bb_middleband_default']) &
                (dataframe['tema_default'] > dataframe['tema_default'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

!!! Tip "使用上游信号"
    通过设置 `remove_entry_exit_signals=false`，你也可以直接使用生产者的信号。这些信号应该以 `enter_long_default` 的形式可用（假设使用了 `suffix="default"`）——可以作为直接信号使用，也可以作为附加指标使用。