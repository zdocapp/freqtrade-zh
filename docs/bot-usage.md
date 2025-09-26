# 启动机器人

本页说明机器人的不同参数及其运行方式。

!!! Note
    如果使用过 `setup.sh`，在运行 freqtrade 命令前请务必激活虚拟环境（`source .venv/bin/activate`）。

!!! Warning "系统时钟同步"
    运行机器人的系统时钟必须准确，需频繁与 NTP 服务器同步以避免与交易所通信出现问题。

## 机器人命令

--8<-- "commands/main.md"

### 机器人交易命令

--8<-- "commands/trade.md"

### 如何指定使用的配置文件？

机器人允许您通过 `-c/--config` 命令行选项选择要使用的配置文件：

```bash
freqtrade trade -c path/far/far/away/config.json
```

默认情况下，机器人会从当前工作目录加载 `config.json` 配置文件。

### 如何使用多个配置文件？

机器人允许您在命令行中指定多个 `-c/--config` 选项来使用多个配置文件。后指定的配置文件中定义的参数会覆盖先前命令行中指定的同名参数。

例如，您可以创建一个独立的配置文件，其中包含您用于交易的交易所密钥和密钥，在运行模拟模式（实际上不需要这些密钥）时指定一个带有空密钥和密钥值的默认配置文件：

```bash
freqtrade trade -c ./config.json
```

并在正常运行的真实交易模式下同时指定这两个配置文件：

```bash
freqtrade trade -c ./config.json -c path/to/secrets/keys.config.json
```

这可以帮助您通过为包含实际密钥的文件设置适当的文件权限来隐藏本地机器上的私有交易所密钥和交易所密钥，此外还能防止在项目问题或互联网上发布配置示例时意外泄露敏感的私有数据。

有关此技术的更多详细信息和示例，请参阅文档页面中的[配置](configuration.md)部分。

### 自定义数据存储位置

Freqtrade 允许使用 `freqtrade create-userdir --userdir someDirectory` 命令创建用户数据目录。该目录结构如下所示：

```
user_data/
├── backtest_results
├── data
├── hyperopts
├── hyperopt_results
├── plot
└── strategies
```

您可以在配置中添加 "user_data_dir" 设置项，使机器人始终指向该目录。或者，在每个命令中传入 `--userdir` 参数。如果目录不存在，机器人将无法启动，但会创建必要的子目录。

此目录应包含您的自定义策略、自定义超参数优化和超参数优化损失函数、回测历史数据（使用回测命令或下载脚本获取）以及绘图输出。

建议使用版本控制来跟踪策略的变更。

### 如何使用 **--strategy** 参数？

该参数允许您加载自定义策略类。
要测试机器人安装，可以使用 `create-userdir` 子命令安装的 `SampleStrategy`（通常位于 `user_data/strategy/sample_strategy.py`）。

机器人将在 `user_data/strategies` 目录中搜索您的策略文件。
如需使用其他目录，请阅读下一节关于 `--strategy-path` 的说明。

加载策略时，只需在此参数中传入类名（例如：`CustomStrategy`）。

**示例：**
在 `user_data/strategies` 目录中，您有一个名为 `my_awesome_strategy.py` 的文件，其中包含名为 `AwesomeStrategy` 的策略类，加载方式如下：

```bash
freqtrade trade --strategy AwesomeStrategy
```

如果机器人找不到您的策略文件，将在错误信息中显示具体原因（文件未找到或代码错误）。

了解更多关于策略文件的信息，请参阅
[策略自定义](strategy-customization.md)。

### 如何使用 **--strategy-path** 参数？

此参数允许您添加额外的策略查找路径，该路径会在默认位置之前被检查（传入的路径必须是目录！）：

```bash
freqtrade trade --strategy AwesomeStrategy --strategy-path /some/directory
```

#### 如何安装策略？

这非常简单。将您的策略文件复制粘贴到 `user_data/strategies` 目录中，或使用 `--strategy-path` 参数。然后，机器人就可以使用它了。

### 如何使用 **--db-url**？

当您在模拟交易模式下运行机器人时，默认情况下不会将交易记录存储在数据库中。如果您希望将机器人操作存储到数据库，可以使用 `--db-url` 参数。这也可用于在生产模式下指定自定义数据库。示例命令：

```bash
freqtrade trade -c config.json --db-url sqlite:///tradesv3.dry_run.sqlite
```

## 下一步

机器人的最优策略会随着市场趋势的变化而改变。下一步是进行[策略定制](strategy-customization.md)。