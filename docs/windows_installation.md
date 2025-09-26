# Windows 安装指南

我们**强烈**建议 Windows 用户使用 [Docker](docker_quickstart.md) 方案，因为这种方式会更加简便顺畅（同时也更安全）。

如果无法使用 Docker，请尝试使用 Windows Linux 子系统 (WSL) - 适用于该环境的 Ubuntu 安装说明应该可以正常工作。
否则，请按照以下说明进行操作。

所有操作指南均假设已安装并可使用 Python 3.11+ 版本。

## 克隆 git 仓库

首先通过以下命令克隆仓库：

``` powershell
git clone https://github.com/freqtrade/freqtrade.git
```

现在请选择您的安装方式，可以通过脚本自动安装（推荐）或按照对应说明手动安装。

## 自动安装 freqtrade

### 运行安装脚本

该脚本将询问几个问题以确定需要安装哪些组件。

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass
cd freqtrade
. .\setup.ps1
```

## 手动安装 freqtrade

!!! Note "64位 Python 版本"
    请确保使用 64 位 Windows 系统和 64 位 Python，以避免因 32 位应用程序在 Windows 下的内存限制而导致回测或超参数优化出现问题。
    Windows 系统下已不再支持 32 位 Python 版本。

!!! Hint
    在 Windows 下使用 [Anaconda 发行版](https://www.anaconda.com/distribution/) 可以极大帮助解决安装问题。查看文档中的 [Anaconda 安装章节](installation.md#installation-with-conda) 了解更多信息。

### Windows 安装过程中的错误

``` bash
error: Microsoft Visual C++ 14.0 is required. Get it with "Microsoft Visual C++ Build Tools": http://landinghub.visualstudio.com/visual-cpp-build-tools
```

遗憾的是，许多需要编译的软件包并未提供预构建的wheel包。因此，必须为您的Python环境安装并配置C/C++编译器方可使用。

您可以从[此处](https://visualstudio.microsoft.com/visual-cpp-build-tools/)下载Visual C++生成工具，并以默认配置安装"使用C++的桌面开发"组件。但需注意，该安装包体积较大/依赖较多，建议您优先考虑使用WSL2或[docker compose](docker_quickstart.md)方案。

![Windows installation](assets/windows_install.png)

---