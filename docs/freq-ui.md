# FreqUI

Freqtrade 提供了一个内置的 Web 服务器，可以运行 [FreqUI](https://github.com/freqtrade/frequi)，即 freqtrade 的前端界面。

默认情况下，UI 会在安装过程中（通过脚本或 Docker）自动安装。
也可以通过手动执行 `freqtrade install-ui` 命令来安装 FreqUI。
该命令同样可用于将 FreqUI 更新到新版本。

当机器人以交易/模拟模式启动（使用 `freqtrade trade` 命令）后，UI 将在配置的 API 端口下可用（默认为 `http://127.0.0.1:8080`）。

??? Note "想要为 FreqUI 做贡献？"
    开发者不应使用此方法，而应按照 [FreqUI 代码库](https://github.com/freqtrade/frequi) 中描述的方法克隆并获取 FreqUI 的源代码。构建前端需要安装可用的 node 环境。

!!! tip "运行 Freqtrade 并不需要 FreqUI"
    FreqUI 是 Freqtrade 的可选组件，并非运行机器人所必需。
    它是一个可用于监控机器人并与之交互的前端界面，但 Freqtrade 本身在没有它的情况下也能完美运行。

## 配置

FreqUI 没有自己的配置文件，但需要确保 [REST API](rest-api.md) 已正确设置并可用。
请参考相应的文档页面来配置 FreqUI。

## 用户界面

FreqUI 是一款现代化的响应式 Web 应用程序，可用于监控您的交易机器人并与之交互。

FreqUI 提供浅色和深色两种主题。
可通过页面顶部的显眼按钮轻松切换主题。
本页截图的主题将适配文档所选主题，因此如需查看深色（或浅色）版本，请切换文档主题。

### 登录界面

下图展示了 FreqUI 的登录界面。

![FreqUI - login](assets/frequi-login-CORS.png#only-dark)
![FreqUI - login](assets/frequi-login-CORS-light.png#only-light)

!!! Hint "CORS"
    此截图中显示的 CORS 错误是由于 UI 与 API 运行在不同端口，且 [CORS](#cors) 尚未正确配置所致。

### 交易视图

交易视图可让您可视化机器人的交易动态并与之交互。
在此页面，您还可以通过启动/停止机器人来与之交互，若已配置，还可强制执行交易入场和出场。

![FreqUI - trade view](assets/freqUI-trade-pane-dark.png#only-dark)
![FreqUI - trade view](assets/freqUI-trade-pane-light.png#only-light)

### 图表配置器

FreqUI 绘图功能可通过策略中的 `plot_config` 配置对象（可通过"从策略加载"按钮载入）或直接通过界面进行配置。
您可以创建多个绘图配置并随意切换，从而灵活地以不同视角查看图表。

绘图配置可通过交易视图右上角的"绘图配置器"（齿轮图标）按钮进行访问。

![FreqUI - 绘图配置](assets/freqUI-plot-configurator-dark.png#only-dark)
![FreqUI - 绘图配置](assets/freqUI-plot-configurator-light.png#only-light)

### 设置

通过访问设置页面可修改多项界面相关设置。

可调整项目包括（但不限于）：

* 界面时区设置
* 在网站图标（浏览器标签页）中显示未平仓交易
* K线颜色配置（上涨/下跌 → 红/绿）
* 启用/禁用应用内通知类型

![FreqUI - 设置界面](assets/frequi-settings-dark.png#only-dark)
![FreqUI - 设置界面](assets/frequi-settings-light.png#only-light)

## 网页服务器模式

当 freqtrade 以[网页服务器模式](utils.md#webserver-mode)启动（使用 `freqtrade webserver` 命令）时，服务器将启动特殊模式以支持额外功能，例如：

* 数据下载
* 交易对列表测试
* [策略回测](#backtesting)
* ... 功能将持续扩展

### 回测

当 freqtrade 以 [webserver 模式](utils.md#webserver-mode) 启动时（使用 `freqtrade webserver` 命令启动），回测视图将变为可用。
该视图允许您对策略进行回测并可视化结果。

您还可以加载并可视化之前的回测结果，以及相互比较这些结果。

![FreqUI - 回测界面](assets/freqUI-backtesting-dark.png#only-dark)
![FreqUI - 回测界面](assets/freqUI-backtesting-light.png#only-light)

--8<-- "includes/cors.md"