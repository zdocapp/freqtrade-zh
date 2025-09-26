# 使用 Jupyter Notebook 分析交易机器人数据

您可以使用 Jupyter Notebook 轻松分析回测结果和交易历史。使用 `freqtrade create-userdir --userdir user_data` 初始化用户目录后，示例 Notebook 位于 `user_data/notebooks/` 目录中。

## Docker 快速入门

Freqtrade 提供了一个 docker-compose 文件，用于启动 Jupyter Lab 服务器。
您可以使用以下命令运行此服务器：`docker compose -f docker/docker-compose-jupyter.yml up`

这将创建一个运行 Jupyter Lab 的 Docker 容器，可通过 `https://127.0.0.1:8888/lab` 访问。
请使用启动后控制台打印的链接以简化登录过程。

更多信息请参阅 [使用 Docker 进行数据分析](docker_quickstart.md#data-analysis-using-docker-compose) 部分。

### 专业提示

* 查看 [jupyter.org](https://jupyter.org/documentation) 获取使用说明。
* 不要忘记在 conda 或 venv 环境中启动 Jupyter Notebook 服务器，或使用 [nb_conda_kernels](https://github.com/Anaconda-Platform/nb_conda_kernels)*
* 使用前请复制示例 Notebook，以免您的更改在下次 Freqtrade 更新时被覆盖。

### 在系统级 Jupyter 安装中使用虚拟环境

有时可能希望使用系统范围内安装的 Jupyter notebook，并利用虚拟环境中的 jupyter kernel。
这样可以避免在系统中多次安装完整的 jupyter 套件，并提供了一种在任务（freqtrade / 其他分析任务）之间切换的简便方法。

为实现此目的，首先激活您的虚拟环境并运行以下命令：

``` bash
# Activate virtual environment
source .venv/bin/activate

pip install ipykernel
ipython kernel install --user --name=freqtrade
# Restart jupyter (lab / notebook)
# select kernel "freqtrade" in the notebook
```

!!! Note
    本节内容为保持完整性而提供，Freqtrade 团队不会对此设置相关的问题提供全面支持，并会推荐直接在虚拟环境中安装 Jupyter，因为这是启动和运行 jupyter notebooks 的最简单方式。有关此设置的帮助，请参阅 [Project Jupyter](https://jupyter.org/) 的[文档](https://jupyter.org/documentation)或[帮助频道](https://jupyter.org/community)。

!!! Warning
    某些任务在 notebooks 中运行效果不佳。例如，任何使用异步执行的功能对 Jupyter 都是问题。此外，freqtrade 的主要入口点是 shell cli，因此在 notebook 中使用纯 Python 会绕过向辅助函数提供必需对象和参数的参数。您可能需要手动设置这些值或创建所需对象。

## 推荐工作流

| 任务 | 工具 |
  --- | ---
机器人操作 | CLI
重复性任务 | Shell 脚本
数据分析与可视化 | Notebook

1. 使用 CLI 进行

    * 下载历史数据
    * 运行回测
    * 使用实时数据运行
    * 导出结果

1. 将操作收集到 shell 脚本中

    * 保存带参数的复杂命令
    * 执行多步骤操作
    * 自动化测试策略和准备分析数据

1. 使用笔记本进行

    * 数据可视化
    * 数据处理和绘图以生成洞察

## 实用代码片段示例

### 切换到根目录

Jupyter notebooks 从笔记本目录执行。以下代码片段会搜索项目根目录，以保持相对路径的一致性。

```python
import os
from pathlib import Path

# Change directory
# Modify this cell to insure that the output shows the correct path.
# Define all paths relative to the project root shown in the cell output
project_root = "somedir/freqtrade"
i=0
try:
    os.chdir(project_root)
    assert Path('LICENSE').is_file()
except:
    while i<4 and (not Path('LICENSE').is_file()):
        os.chdir(Path(Path.cwd(), '../'))
        i+=1
    project_root = Path.cwd()
print(Path.cwd())
```

### 加载多个配置文件

此选项可用于检查传入多个配置的结果。
这也会运行完整的配置初始化过程，因此配置会完全初始化以便传递给其他方法。

``` python
import json
from freqtrade.configuration import Configuration

# Load config from multiple files
config = Configuration.from_files(["config1.json", "config2.json"])

# Show the config in memory
print(json.dumps(config['original_config'], indent=2))
```

对于交互式环境，可以额外指定 `user_data_dir` 的配置，并最后传入，这样在运行机器人时就不需要更改目录。
最好避免使用相对路径，因为这会从 jupyter notebook 的存储位置开始，除非更改了目录。

``` json
{
    "user_data_dir": "~/.freqtrade/"
}
```

### 进一步的数据分析文档

* [策略调试](strategy_analysis_example.md) - 也可作为 Jupyter notebook 使用 (`user_data/notebooks/strategy_analysis_example.ipynb`)
* [绘图](plotting.md)
* [标签分析](advanced-backtesting.md)

如果您有关于如何最佳分析数据的想法，欢迎提交问题或拉取请求来改进本文档。