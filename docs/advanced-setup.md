# 高级安装后任务

本页介绍了一些在机器人安装后可以执行的高级任务和配置选项，这些内容在某些环境中可能非常有用。

如果您不明白这里提到的事项是什么意思，那么您很可能不需要它们。

## 运行多个 Freqtrade 实例

本节将向您展示如何在同一台机器上同时运行多个机器人。

### 需要考虑的事项

* 使用不同的数据库文件
* 使用不同的 Telegram 机器人（需要多个不同的配置文件；仅当启用 Telegram 时适用）
* 使用不同的端口（仅当启用 Freqtrade REST API 网络服务器时适用）

### 不同的数据库文件

为了跟踪您的交易、利润等，freqtrade 使用 SQLite 数据库存储各种类型的信息，例如您过去执行的交易以及您在任何时候持有的当前头寸。这使您可以跟踪利润，但最重要的是，在机器人进程重新启动或意外终止时跟踪正在进行的活动。

默认情况下，Freqtrade 将为模拟交易和实盘机器人使用单独的数据库文件（这假设在配置中或通过命令行参数均未给出数据库 URL）。
对于实盘交易模式，默认数据库将为 `tradesv3.sqlite`，而对于模拟交易则为 `tradesv3.dryrun.sqlite`。

用于指定这些文件路径的 trade 命令可选参数是 `--db-url`，它需要一个有效的 SQLAlchemy 连接地址。
因此，当你在模拟交易模式下仅使用配置和策略参数启动机器人时，以下两条命令将产生相同的结果。

``` bash
freqtrade trade -c MyConfig.json -s MyStrategy
# is equivalent to
freqtrade trade -c MyConfig.json -s MyStrategy --db-url sqlite:///tradesv3.dryrun.sqlite
```

这意味着如果你在两个不同的终端中运行 trade 命令，例如一个实例测试 USDT 交易策略，另一个实例测试 BTC 交易策略，则需要为它们指定不同的数据库。

如果指定一个不存在的数据库 URL，freqtrade 将使用你指定的名称创建一个新数据库。因此要使用 BTC 和 USDT 作为本位币测试自定义策略，可以运行以下命令（在两个独立终端中）：

``` bash
# Terminal 1:
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.dryrun.sqlite
# Terminal 2:
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.dryrun.sqlite
```

相反，如果要在实盘模式下执行相同操作，除了默认数据库外至少还需要创建一个新数据库，并指定"实盘"数据库的路径，例如：

``` bash
# Terminal 1:
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.live.sqlite
# Terminal 2:
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.live.sqlite
```

有关 SQLite 数据库使用的更多信息，例如手动添加或删除交易记录，请参阅 [SQL 速查表](sql_cheatsheet.md)。

### 使用 Docker 运行多实例

要使用 Docker 运行多个 freqtrade 实例，您需要编辑 docker-compose.yml 文件，并将所有需要的实例作为独立服务添加进去。请记住，您可以将配置分离到多个文件中，因此考虑采用模块化设计是个好主意，这样如果需要编辑所有机器人的通用配置，只需在单个配置文件中修改即可。

``` yml
---
version: '3'
services:
  freqtrade1:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # Use plotting image
    # image: freqtradeorg/freqtrade:develop_plot
    # Build step - only needed when additional dependencies are needed
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade1
    volumes:
      - "./user_data:/freqtrade/user_data"
    # Expose api on port 8080 (localhost only)
    # Please read the https://www.freqtrade.io/en/latest/rest-api/ documentation
    # before enabling this.
     ports:
     - "127.0.0.1:8080:8080"
    # Default command used when running `docker compose up`
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade1.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade1.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade1.json
      --strategy SampleStrategy
  
  freqtrade2:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # Use plotting image
    # image: freqtradeorg/freqtrade:develop_plot
    # Build step - only needed when additional dependencies are needed
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade2
    volumes:
      - "./user_data:/freqtrade/user_data"
    # Expose api on port 8080 (localhost only)
    # Please read the https://www.freqtrade.io/en/latest/rest-api/ documentation
    # before enabling this.
    ports:
      - "127.0.0.1:8081:8080"
    # Default command used when running `docker compose up`
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade2.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade2.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade2.json
      --strategy SampleStrategy

```

您可以采用任意命名规范，示例中的 freqtrade1 和 2 仅为示意名称。请注意，如上所述，每个实例需要使用不同的数据库文件、端口映射和 Telegram 配置。

## 使用其他数据库系统

Freqtrade 使用支持多种数据库系统的 SQLAlchemy，因此应兼容大多数数据库系统。
Freqtrade 不依赖也不安装任何额外的数据库驱动。请参阅 [SQLAlchemy 文档](https://docs.sqlalchemy.org/en/14/core/engines.html#database-urls)了解相应数据库系统的安装说明。

以下系统已通过测试并确认可与 freqtrade 协同工作：

* sqlite（默认）
* PostgreSQL
* MariaDB

!!! Warning
    使用以下任意数据库系统即表示您知晓如何管理此类系统。freqtrade 团队不提供以下数据库系统的设置、维护（或备份）相关支持。

### PostgreSQL

安装方法：
`pip install psycopg2-binary`

用法：
`... --db-url postgresql+psycopg2://<用户名>:<密码>@localhost:5432/<数据库>`

Freqtrade 将在启动时自动创建所需的表。

如果您运行多个 Freqtrade 实例，必须为每个实例单独设置数据库，或在连接中使用不同的用户/模式。

### MariaDB / MySQL

Freqtrade 通过 SQLAlchemy 支持 MariaDB，该库兼容多种数据库系统。

安装：
`pip install pymysql`

用法：
`... --db-url mysql+pymysql://<用户名>:<密码>@localhost:3306/<数据库>`

## 配置机器人作为 systemd 服务运行

将 `freqtrade.service` 文件复制到 systemd 用户目录（通常为 `~/.config/systemd/user`），并修改 `WorkingDirectory` 和 `ExecStart` 以匹配您的设置。

!!! Note
    某些系统（如 Raspbian）不会从用户目录加载服务单元文件。此时请将 `freqtrade.service` 复制到 `/etc/systemd/user/`（需要超级用户权限）。

之后您可以通过以下命令启动守护进程：

```bash
systemctl --user start freqtrade
```

如需实现持久化（用户注销后仍运行），需为 freqtrade 用户启用 `linger` 功能。

```bash
sudo loginctl enable-linger "$USER"
```

如果您将机器人作为服务运行，可以使用 systemd 服务管理器作为软件看门狗来监控 freqtrade 机器人的状态，并在发生故障时重启它。如果在配置中设置 `internals.sd_notify` 参数为 true 或使用 `--sd-notify` 命令行选项，机器人将通过 sd_notify（systemd 通知）协议向 systemd 发送存活 ping 消息，并在状态改变时告知 systemd 其当前状态（运行中、暂停或停止）。

`freqtrade.service.watchdog` 文件包含了一个使用 systemd 作为看门狗的服务单元配置文件示例。

!!! Note
    如果机器人在 Docker 容器中运行，机器人与 systemd 服务管理器之间的 sd_notify 通信将无法工作。

## 高级日志配置

Freqtrade 使用 Python 提供的默认日志记录模块。
Python 在这方面支持广泛的[日志配置](https://docs.python.org/3/library/logging.config.html#logging.config.dictConfig) - 远超本文所能涵盖的范围。

如果在您的 freqtrade 配置中未提供 `log_config`，默认会设置默认日志格式（彩色终端输出）。
使用 `--logfile logfile.log` 将启用 RotatingFileHandler。

如果您对日志格式不满意，或者对 RotatingFileHandler 提供的默认设置不满意，可以通过在 freqtrade 配置文件中添加 `log_config` 配置来自定义日志记录。

默认配置大致如下所示，其中文件处理程序已提供但未启用，因为 `filename` 被注释掉了。取消注释该行并提供有效的路径/文件名即可启用它。

``` json hl_lines="5-7 13-16 27"
{
  "log_config": {
      "version": 1,
      "formatters": {
          "basic": {
              "format": "%(message)s"
          },
          "standard": {
              "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
          }
      },
      "handlers": {
          "console": {
              "class": "freqtrade.loggers.ft_rich_handler.FtRichHandler",
              "formatter": "basic"
          },
          "file": {
              "class": "logging.handlers.RotatingFileHandler",
              "formatter": "standard",
              // "filename": "someRandomLogFile.log",
              "maxBytes": 10485760,
              "backupCount": 10
          }
      },
      "root": {
          "handlers": [
              "console",
              // "file"
          ],
          "level": "INFO",
      }
  }
}
```

!!! Note "highlighted lines"
    上述代码块中高亮显示的行定义了 Rich 处理程序，它们属于一个整体。
    格式化程序 "standard" 和 "file" 将属于 FileHandler。

每个处理程序必须使用一个已定义的格式化程序（按名称），其类必须可用，并且必须是有效的日志记录类。
要实际使用处理程序，它必须位于 "root" 段内的 "handlers" 部分。
如果省略此部分，freqtrade 将不提供任何输出（至少在未配置的处理程序中）。

!!! Tip "Explicit log configuration"
    我们建议将日志记录配置从主 freqtrade 配置文件中提取出来，并通过[多配置文件](configuration.md#multiple-configuration-files)功能提供给您的机器人。这将避免不必要的代码重复。

---

在许多 Linux 系统上，机器人可以配置为将其日志消息发送到 `syslog` 或 `journald` 系统服务。在 Windows 上也可使用远程 `syslog` 服务器日志记录。为此可以使用 `--logfile` 命令行选项的特殊值。

### 记录到 syslog

要将 Freqtrade 日志消息发送到本地或远程 `syslog` 服务，请使用 `"log_config"` 设置选项来配置日志记录。

``` json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "syslog_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // Other handlers? 
      "syslog": {
         "class": "logging.handlers.SysLogHandler",
          "formatter": "syslog_fmt",
          // Use one of the other options above as address instead? 
          "address": "/dev/log"
      }
    },
    "root": {
      "handlers": [
        // other handlers
        "syslog",
        
      ]
    }

  }
}
```

可能需要配置[额外的日志处理器](#advanced-logging)，例如同时在控制台输出日志。

#### Syslog 用法

日志消息通过 `user` 设施发送到 `syslog`。因此您可以通过以下命令查看它们：

* `tail -f /var/log/user`，或
* 安装一个综合的图形化查看器（例如，Ubuntu 的“日志文件查看器”）。

在许多系统上，`syslog`（`rsyslog`）会从 `journald` 获取数据（反之亦然），因此可以同时使用 syslog 或 journald，并通过 `journalctl` 和 syslog 查看器工具查看消息。您可以根据需要以任何方式组合使用。

对于 `rsyslog`，来自机器人的消息可以重定向到单独的专用日志文件。要实现这一点，请将

```
if $programname startswith "freqtrade" then -/var/log/freqtrade.log
```

添加到某个 rsyslog 配置文件中，例如在 `/etc/rsyslog.d/50-default.conf` 的末尾。

对于 `syslog`（`rsyslog`），可以开启精简模式。这将减少重复消息的数量。例如，当机器人没有其他活动时，多个机器人心跳消息将被精简为单条消息。要实现此功能，请在 `/etc/rsyslog.conf` 中设置：

```
# Filter duplicated messages
$RepeatedMsgReduction on
```

#### Syslog 寻址

syslog 地址可以是 Unix 域套接字（套接字文件名）或 UDP 套接字规范，由 IP 地址和 UDP 端口组成，以 `:` 字符分隔。

因此，以下是可能的地址示例：

* `"address": "/dev/log"` -- 使用 `/dev/log` 套接字记录到 syslog（rsyslog），适用于大多数系统。
* `"address": "/var/run/syslog"` -- 使用 `/var/run/syslog` 套接字记录到 syslog（rsyslog）。在 MacOS 上使用此地址。
* `"address": "localhost:514"` -- 如果本地 syslog 监听 514 端口，则使用 UDP 套接字记录到本地 syslog。
* `"address": "<ip>:514"` -- 记录到指定 IP 地址和 514 端口的远程 syslog。在 Windows 上可用于远程记录到外部 syslog 服务器。

??? 信息 "已弃用 - 通过命令行配置 syslog"
    `--logfile syslog:<syslog_address>` -- 使用 `<syslog_address>` 作为 syslog 地址，将日志消息发送到 `syslog` 服务。

    The syslog address can be either a Unix domain socket (socket filename) or a UDP socket specification, consisting of IP address and UDP port, separated by the `:` character.

    So, the following are the examples of possible usages:

    * `--logfile syslog:/dev/log` -- log to syslog (rsyslog) using the `/dev/log` socket, suitable for most systems.
    * `--logfile syslog` -- same as above, the shortcut for `/dev/log`.
    * `--logfile syslog:/var/run/syslog` -- log to syslog (rsyslog) using the `/var/run/syslog` socket. Use this on MacOS.
    * `--logfile syslog:localhost:514` -- log to local syslog using UDP socket, if it listens on port 514.
    * `--logfile syslog:<ip>:514` -- log to remote syslog at IP address and port 514. This may be used on Windows for remote logging to an external syslog server.

### 记录到 journald

这需要安装 `cysystemd` python 包作为依赖项（`pip install cysystemd`），但该包在 Windows 上不可用。因此，在 Windows 上运行的机器人无法使用整个 journald 日志记录功能。

要将 Freqtrade 日志消息发送到 `journald` 系统服务，请将以下配置片段添加到您的配置中。

``` json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "journald_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // Other handlers? 
      "journald": {
         "class": "cysystemd.journal.JournaldLogHandler",
          "formatter": "journald_fmt",
      }
    },
    "root": {
      "handlers": [
        // .. 
        "journald",
        
      ]
    }

  }
}
```

可能需要配置[额外的日志处理器](#advanced-logging)，例如同时在控制台输出日志。

日志消息以 `user` 设施类型发送到 `journald`。因此您可以通过以下命令查看它们：

* `journalctl -f` -- 显示发送到 `journald` 的 Freqtrade 日志消息以及 `journald` 获取的其他日志消息。
* `journalctl -f -u freqtrade.service` -- 当机器人作为 `systemd` 服务运行时可以使用此命令。

`journalctl` 实用程序中有许多其他选项可用于过滤消息，请参阅该实用程序的手册页。

在许多系统上，`syslog`（`rsyslog`）会从 `journald` 获取数据（反之亦然），因此可以同时使用 `--logfile syslog` 或 `--logfile journald`，并且可以通过 `journalctl` 和 syslog 查看器实用程序查看消息。您可以根据需要以任何方式组合使用。

??? 信息 "已弃用 - 通过命令行配置 journald"
    要通过命令行将 Freqtrade 日志消息发送到 `journald` 系统服务，请使用 `--logfile` 命令行选项，并按以下格式指定值：

    `--logfile journald` -- send log messages to `journald`.

### 日志格式设为 JSON

您也可以将默认输出流配置为使用 JSON 格式。
"fmt_dict" 属性定义了 JSON 输出的键名 - 以及 [python logging LogRecord 属性](https://docs.python.org/3/library/logging.html#logrecord-attributes)。

以下配置将把默认输出更改为 JSON。不过，相同的格式化器也可以与 `RotatingFileHandler` 结合使用。
我们建议保留一种人类可读的格式。

``` json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
       "json": {
          "()": "freqtrade.loggers.json_formatter.JsonFormatter",
          "fmt_dict": {
              "timestamp": "asctime",
              "level": "levelname",
              "logger": "name",
              "message": "message"
          }
      }
    },
    "handlers": {
      // Other handlers? 
      "jsonStream": {
          "class": "logging.StreamHandler",
          "formatter": "json"
      }
    },
    "root": {
      "handlers": [
        // .. 
        "jsonStream",
        
      ]
    }

  }
}
```