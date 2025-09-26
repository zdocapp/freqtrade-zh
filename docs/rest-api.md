# REST API

## FreqUI

FreqUI 现在拥有独立的[文档章节](freq-ui.md) - 有关 FreqUI 的所有信息请参考该章节。

## 配置

通过在配置中添加 api_server 部分并将 `api_server.enabled` 设置为 `true` 来启用 REST API。

示例配置：

``` json
    "api_server": {
        "enabled": true,
        "listen_ip_address": "127.0.0.1",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "somethingrandom",
        "CORS_origins": [],
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        "ws_token": "sercet_Ws_t0ken"
    },
```

!!! Danger "安全警告"
    默认情况下，配置仅监听本地主机（因此无法从其他系统访问）。我们强烈建议不要将此 API 暴露到互联网上，并选择强且唯一的密码，因为其他人可能能够控制您的机器人。

??? Note "远程服务器上的 API/UI 访问"
    如果您在 VPS 上运行，应考虑使用 SSH 隧道或设置 VPN（openVPN、wireguard）来连接您的机器人。
    这将确保 freqUI 不会直接暴露在互联网上，出于安全原因不建议这样做（freqUI 不支持开箱即用的 https）。
    这些工具的设置不在本教程范围内，但可以在互联网上找到许多优秀的教程。

然后您可以通过在浏览器中访问 `http://127.0.0.1:8080/api/v1/ping` 来检查 API 是否正常运行。
这应该返回响应：

``` output
{"status":"pong"}
```

所有其他端点返回敏感信息，需要身份验证，因此无法通过 Web 浏览器访问。

### 安全性

要生成安全密码，最佳方式是使用密码管理器，或使用以下代码。

``` python
import secrets
secrets.token_hex()
```

!!! Hint "JWT token"
    使用相同方法同时生成 JWT 密钥（`jwt_secret_key`）。

!!! Danger "Password selection"
    请务必选择一个非常强大且唯一的密码，以保护您的交易机器人免受未经授权的访问。
    同时将 `jwt_secret_key` 更改为随机值（无需记住此密钥，但它将用于加密您的会话，因此最好使用唯一值！）。

### Docker 配置

如果您使用 docker 运行交易机器人，需要让机器人监听传入连接。此时安全由 docker 处理。

``` json
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        //...
    },
```

请确保您的 docker-compose 文件中包含以下两行内容：

```yml
    ports:
      - "127.0.0.1:8080:8080"
```

!!! Danger "Security warning"
    在 docker 端口映射中使用 `"8080:8080"`（或 `"0.0.0.0:8080:8080"`）将使 API 对连接到服务器正确端口的所有人可用，因此其他人可能能够控制您的交易机器人。
    如果在安全环境（如家庭网络）中运行机器人**可能**安全，但不建议将 API 暴露到互联网。

## Rest API

### 使用 API

我们建议使用受支持的 `freqtrade-client` 包（也可作为 `scripts/rest_client.py` 使用）来调用 API。

该命令可通过 `pip install freqtrade-client` 独立于任何正在运行的 freqtrade 机器人进行安装。

该模块设计为轻量级，仅依赖 `requests` 和 `python-rapidjson` 模块，跳过了 freqtrade 原本所需的所有重型依赖。

``` bash
freqtrade-client <command> [optional parameters]
```

默认情况下，脚本假定使用 `127.0.0.1`（本地主机）和端口 `8080`，但您可以通过指定配置文件来覆盖此行为。

#### 极简客户端配置

``` json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        //...
    }
}
```

``` bash
freqtrade-client --config rest_config.json <command> [optional parameters]
```

包含多个参数的命令可能需要使用关键字参数（为了清晰起见）——可按如下方式提供：

``` bash
freqtrade-client --config rest_config.json forceenter BTC/USDT long enter_tag=GutFeeling
```

此方法适用于所有参数——请查看 "show" 命令以获取可用参数列表。

??? Note "编程式使用"
    `freqtrade-client` 包（可独立于 freqtrade 安装）可在您自己的脚本中用于与 freqtrade API 交互。
    为此，请使用以下方式：

    ``` python
    from freqtrade_client import FtRestClient
    

    client = FtRestClient(server_url, username, password)

    # Get the status of the bot
    ping = client.ping()
    print(ping)

    # Add pairs to blacklist
    client.blacklist("BTC/USDT", "ETH/USDT")
    # Add pairs to blacklist by supplying a list
    client.blacklist(*listPairs)
    # ... 
    ```

    For a full list of available commands, please refer to the list below.

可通过 rest-client 脚本使用 `help` 命令列出可能的命令。

``` bash
freqtrade-client help
```

``` output
Possible commands:

available_pairs
    Return available pair (backtest data) based on timeframe / stake_currency selection

        :param timeframe: Only pairs with this timeframe available.
        :param stake_currency: Only pairs that include this timeframe

balance
    Get the account balance.

blacklist
    Show the current blacklist.

        :param add: List of coins to add (example: "BNB/BTC")

cancel_open_order
    Cancel open order for trade.

        :param trade_id: Cancels open orders for this trade.

count
    Return the amount of open trades.

daily
    Return the profits for each day, and amount of trades.

delete_lock
    Delete (disable) lock from the database.

        :param lock_id: ID for the lock to delete

delete_trade
    Delete trade from the database.
        Tries to close open orders. Requires manual handling of this asset on the exchange.

        :param trade_id: Deletes the trade with this ID from the database.

forcebuy
    Buy an asset.

        :param pair: Pair to buy (ETH/BTC)
        :param price: Optional - price to buy

forceenter
    Force entering a trade

        :param pair: Pair to buy (ETH/BTC)
        :param side: 'long' or 'short'
        :param price: Optional - price to buy

forceexit
    Force-exit a trade.

        :param tradeid: Id of the trade (can be received via status command)
        :param ordertype: Order type to use (must be market or limit)
        :param amount: Amount to sell. Full sell if not given

health
    Provides a quick health check of the running bot.

lock_add
    Manually lock a specific pair

        :param pair: Pair to lock
        :param until: Lock until this date (format "2024-03-30 16:00:00Z")
        :param side: Side to lock (long, short, *)
        :param reason: Reason for the lock        

locks
    Return current locks

logs
    Show latest logs.

        :param limit: Limits log messages to the last <limit> logs. No limit to get the entire log.

pair_candles
    Return live dataframe for <pair><timeframe>.

        :param pair: Pair to get data for
        :param timeframe: Only pairs with this timeframe available.
        :param limit: Limit result to the last n candles.

pair_history
    Return historic, analyzed dataframe

        :param pair: Pair to get data for
        :param timeframe: Only pairs with this timeframe available.
        :param strategy: Strategy to analyze and get values for
        :param timerange: Timerange to get data for (same format than --timerange endpoints)

performance
    Return the performance of the different coins.

ping
    simple ping

plot_config
    Return plot configuration if the strategy defines one.

profit
    Return the profit summary.

reload_config
    Reload configuration.

show_config
    Returns part of the configuration, relevant for trading operations.

start
    Start the bot if it's in the stopped state.

pause
    Pause the bot if it's in the running state. If triggered on stopped state will handle open positions.

stats
    Return the stats report (durations, sell-reasons).

status
    Get the status of open trades.

stop
    Stop the bot. Use `start` to restart.

stopbuy
    Stop buying (but handle sells gracefully). Use `reload_config` to reset.

strategies
    Lists available strategies

strategy
    Get strategy details

        :param strategy: Strategy class name

sysinfo
    Provides system information (CPU, RAM usage)

trade
    Return specific trade

        :param trade_id: Specify which trade to get.

trades
    Return trades history, sorted by id

        :param limit: Limits trades to the X last trades. Max 500 trades.
        :param offset: Offset by this amount of trades.

list_open_trades_custom_data
    Return a dict containing open trades custom-datas

        :param key: str, optional - Key of the custom-data
        :param limit: Limits trades to X trades.
        :param offset: Offset by this amount of trades.

list_custom_data
    Return a dict containing custom-datas of a specified trade

        :param trade_id: int - ID of the trade
        :param key: str, optional - Key of the custom-data

version
    Return the version of the bot.

whitelist
    Show the current whitelist.


```

### 可用端点

如果您希望通过其他途径手动调用 REST API，例如直接通过 `curl`，下表显示了相关的 URL 端点和参数。
下表中的所有端点都需要以 API 的基础 URL 为前缀，例如 `http://127.0.0.1:8080/api/v1/`——因此命令变为 `http://127.0.0.1:8080/api/v1/<command>`。

| 端点 | 方法 | 描述 / 参数 |
|-----------|--------|--------------------------|
| `/ping` | GET | 简单的 API 就绪性测试命令 - 无需身份验证。
| `/start` | POST | 启动交易机器人。
| `/pause` | POST | 暂停交易机器人。根据其规则妥善处理未平仓交易。不建立新仓位。
| `/stop` | POST | 停止交易机器人。
| `/stopbuy` | POST | 停止交易机器人开立新交易。根据其规则妥善平仓未平仓交易。
| `/reload_config` | POST | 重新加载配置文件。
| `/trades` | GET | 列出最近的交易。每次调用限制为 500 笔交易。
| `/trade/<tradeid>` | GET | 获取特定交易。<br/>*参数:*<br/>- `tradeid` (`int`)
| `/trades/<tradeid>` | DELETE | 从数据库中移除交易。尝试取消未成交订单。需要在交易所手动处理此交易。<br/>*参数:*<br/>- `tradeid` (`int`)
| `/trades/<tradeid>/open-order` | DELETE | 取消此交易的未成交订单。<br/>*参数:*<br/>- `tradeid` (`int`)
| `/trades/<tradeid>/reload` | POST | 从交易所重新加载交易。仅在生产环境有效，可能有助于恢复在交易所手动卖出的交易。<br/>*参数:*<br/>- `tradeid` (`int`)
| `/show_config` | GET | 显示当前配置中与操作相关的部分设置。
| `/logs` | GET | 显示最新的日志消息。
| `/status` | GET | 列出所有未平仓交易。
| `/count` | GET | 显示已使用和可用的交易数量。
| `/entries` | GET | 显示给定交易对（如果未指定交易对则为所有交易对）的每个入场标签的利润统计信息。交易对为可选参数。<br/>*参数:*<br/>- `pair` (`str`)
| `/exits` | GET | 显示给定交易对（如果未指定交易对则为所有交易对）的每个离场原因的利润统计信息。交易对为可选参数。<br/>*参数:*<br/>- `pair` (`str`)
| `/mix_tags` | GET | 显示给定交易对（如果未指定交易对则为所有交易对）的每个入场标签 + 离场原因组合的利润统计信息。交易对为可选参数。<br/>*参数:*<br/>- `pair` (`str`)
| `/locks` | GET | 显示当前被锁定的交易对。
| `/locks` | POST | 锁定一个交易对直到 "until"（"until" 将向上取整到最近的时间框架）。方向为可选参数，可以是 `long` 或 `short`（默认为 `long`）。原因为可选参数。<br/>*参数:*<br/>- `<pair>` (`str`)<br/>- `<until>` (`datetime`)<br/>- `[side]` (`str`)<br/>- `[reason]` (`str`)
| `/locks/<lockid>` | DELETE | 根据 ID 删除（禁用）锁定。<br/>*参数:*<br/>- `lockid` (`int`)
| `/profit` | GET | 显示已平仓交易的盈亏摘要以及一些关于您表现的数据统计。
| `/forceexit` | POST | 立即退出给定交易（忽略 `minimum_roi`），使用给定的订单类型（"market" 或 "limit"，如果未指定则使用您的配置设置），以及选定的数量（如果未指定则为全仓卖出）。如果将 `all` 作为 `tradeid` 提供，则所有当前未平仓交易将被强制退出。<br/>*参数:*<br/>- `<tradeid>` (`int` 或 `str`)<br/>- `<ordertype>` (`str`)<br/>- `[amount]` (`float`)
| `/forceenter` | POST | 立即进入给定交易对。方向为可选参数，可以是 `long` 或 `short`（默认为 `long`）。价格为可选参数。（必须将 `force_entry_enable` 设置为 True）<br/>*参数:*<br/>- `<pair>` (`str`)<br/>- `<side>` (`str`)<br/>- `[rate]` (`float`)
| `/performance` | GET | 显示按交易对分组的所有已完成交易的绩效。
| `/balance` | GET | 显示每种货币的账户余额。
| `/daily` | GET | 显示过去 n 天（n 默认为 7）每天的利润或损失。<br/>*参数:*<br/>- `timescale` (`int`)
| `/weekly` | GET | 显示过去 n 天（n 默认为 4）每周的利润或损失。<br/>*参数:*<br/>- `timescale` (`int`)
| `/monthly` | GET | 显示过去 n 天（n 默认为 3）每月的利润或损失。<br/>*参数:*<br/>- `timescale` (`int`)
| `/stats` | GET | 显示盈亏原因以及平均持仓时间的摘要。
| `/whitelist` | GET | 显示当前白名单。
| `/blacklist` | GET | 显示当前黑名单。
| `/blacklist` | POST | 将指定交易对添加到黑名单。<br/>*参数:*<br/>- `blacklist` (`str`)
| `/blacklist` | DELETE | 从黑名单中删除指定的交易对列表。<br/>*参数:*<br/>- `[pair,pair]` (`list[str]`)
| `/pair_candles` | GET | 在机器人运行时返回某个交易对/时间框架组合的数据框。**Alpha**
| `/pair_candles` | POST | 在机器人运行时返回某个交易对/时间框架组合的数据框，并通过提供的列列表进行过滤返回。**Alpha**<br/>*参数:*<br/>- `<column_list>` (`list[str]`)
| `/pair_history` | GET | 返回给定时间范围内、由给定策略分析后的已分析数据框。**Alpha**
| `/pair_history` | POST | 返回给定时间范围内、由给定策略分析后的已分析数据框，并通过提供的列列表进行过滤返回。**Alpha**<br/>*参数:*<br/>- `<column_list>` (`list[str]`)
| `/plot_config` | GET | 从策略获取绘图配置（如果未配置则返回空）。**Alpha**
| `/strategies` | GET | 列出策略目录中的策略。**Alpha**
| `/strategy/<strategy>` | GET | 通过策略类名获取特定策略内容。**Alpha**<br/>*参数:*<br/>- `<strategy>` (`str`)
| `/available_pairs` | GET | 列出可用的回测数据。**Alpha**
| `/version` | GET | 显示版本信息。
| `/sysinfo` | GET | 显示系统负载信息。
| `/health` | GET | 显示机器人健康状况（最近一次机器人循环）。

!!! Warning "Alpha status"
    上方标记为 *Alpha status* 的端点可能随时更改，恕不另行通知。

### 消息 WebSocket

API 服务器包含一个 WebSocket 端点，用于订阅来自 Freqtrade 机器人的 RPC 消息。
这可用于消费来自机器人的实时数据，例如入场/出场成交消息、白名单变更、交易对的填充指标等。

这也用于在 Freqtrade 中设置 [生产者/消费者模式](producer-consumer.md)。

假设您的 REST API 设置为 `127.0.0.1`，端口为 `8080`，则该端点位于 `http://localhost:8080/api/v1/message/ws`。

要访问 WebSocket 端点，需要在端点 URL 中将 `ws_token` 作为查询参数提供。

要生成安全的 `ws_token`，您可以运行以下代码：

``` python
>>> import secrets
>>> secrets.token_urlsafe(25)
'hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q'
```

然后，您需要将该令牌添加到 `api_server` 配置中的 `ws_token` 下。如下所示：

``` json
"api_server": {
    "enabled": true,
    "listen_ip_address": "127.0.0.1",
    "listen_port": 8080,
    "verbosity": "error",
    "enable_openapi": false,
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": [],
    "username": "Freqtrader",
    "password": "SuperSecret1!",
    "ws_token": "hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q" // <-----
},
```

您现在可以连接到端点 `http://localhost:8080/api/v1/message/ws?token=hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q`。

!!! Danger "重复使用示例令牌"
    请勿使用上述示例令牌。为确保安全，请生成一个全新的令牌。

#### 使用 WebSocket

连接到 WebSocket 后，机器人将向所有订阅者广播 RPC 消息。要订阅消息列表，您必须通过 WebSocket 发送如下所示的 JSON 请求。`data` 键必须是一个消息类型字符串列表。

``` json
{
  "type": "subscribe",
  "data": ["whitelist", "analyzed_df"] // A list of string message types
}
```

有关消息类型的列表，请参阅 `freqtrade/enums/rpcmessagetype.py` 中的 RPCMessageType 枚举

现在只要连接处于活动状态，每当机器人发送这些类型的 RPC 消息时，您都会通过 WebSocket 接收到它们。这些消息通常与请求采用相同的形式：

``` json
{
  "type": "analyzed_df",
  "data": {
      "key": ["NEO/BTC", "5m", "spot"],
      "df": {}, // The dataframe
      "la": "2022-09-08 22:14:41.457786+00:00"
  }
}
```

#### 反向代理设置

使用 [Nginx](https://nginx.org/en/docs/) 时，需要以下配置才能使 WebSocket 正常工作（请注意此配置不完整，缺少一些信息，不能直接使用）：

请确保将 `<freqtrade_listen_ip>`（及后续端口）替换为与您的配置/设置匹配的 IP 和端口。

```
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    #...

    server {
        #...

        location / {
            proxy_http_version 1.1;
            proxy_pass http://<freqtrade_listen_ip>:8080;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
        }
    }
}
```

要正确（安全地）配置反向代理，请查阅其代理 WebSocket 的文档。

- **Traefik**: Traefik 开箱即用支持 WebSocket，请参阅 [文档](https://doc.traefik.io/traefik/)
- **Caddy**: Caddy v2 开箱即用支持 WebSocket，请参阅 [文档](https://caddyserver.com/docs/v2-upgrade#proxy)

!!! Tip "SSL certificates"
    您可以使用 certbot 等工具设置 SSL 证书，通过上述任一反向代理以加密连接访问您的交易机器人界面。
    虽然这会保护传输中的数据，但我们不建议在私有网络（VPN、SSH 隧道）之外运行 freqtrade API。

### OpenAPI 接口

要启用内置的 OpenAPI 接口（Swagger UI），请在 api_server 配置中指定 `"enable_openapi": true`。
这将启用 `/docs` 端点上的 Swagger UI。默认情况下，它运行在 <http://localhost:8080/docs> - 但这取决于您的设置。

### 使用 JWT 令牌的高级 API 用法

!!! Note
    以下操作应在应用程序（通过 API 获取信息的 Freqtrade REST API 客户端）中完成，不适用于常规使用。

Freqtrade 的 REST API 还提供 JWT（JSON Web 令牌）。
您可以使用以下命令登录，并随后使用生成的 access_token。

``` bash
> curl -X POST --user Freqtrader http://localhost:8080/api/v1/token/login
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiZWQ1ZWI3YjAtYjMwMy00YzAyLTg2N2MtNWViMjIxNWQ2YTMxIiwiZXhwIjoxNTkxNzExNjgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJ0eXBlIjoicmVmcmVzaCJ9.d1AT_jYICyTAjD0fiQAr52rkRqtxCjUGEMwlNuuzgNQ"}

> access_token="eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g"
# Use access_token for authentication
> curl -X GET --header "Authorization: Bearer ${access_token}" http://localhost:8080/api/v1/count

```

由于访问令牌具有较短的超时时间（15 分钟）- 应定期使用 `token/refresh` 请求来获取新的访问令牌：

``` bash
> curl -X POST --header "Authorization: Bearer ${refresh_token}"http://localhost:8080/api/v1/token/refresh
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk5NzQsIm5iZiI6MTU4OTExOTk3NCwianRpIjoiMDBjNTlhMWUtMjBmYS00ZTk0LTliZjAtNWQwNTg2MTdiZDIyIiwiZXhwIjoxNTg5MTIwODc0LCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.1seHlII3WprjjclY6DpRhen0rqdF4j6jbvxIhUFaSbs"}
```

--8<-- "includes/cors.md"