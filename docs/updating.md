# 如何更新

要更新您的 freqtrade 安装，请根据您的安装方式使用以下对应方法。

!!! Note "跟踪变更"
    重大变更/行为变化将在每个版本随附的更新日志中记录。
    对于 develop 分支，请关注 PR 以避免被变更所影响。

## Docker 部署

!!! Note "使用 `master` 镜像的传统安装"
    我们正在将发布镜像从 master 切换为 stable - 请调整您的 docker 文件，将 `freqtradeorg/freqtrade:master` 替换为 `freqtradeorg/freqtrade:stable`

``` bash
docker compose pull
docker compose up -d
```

## 通过安装脚本安装

``` bash
./setup.sh --update
```

!!! Note
    请确保在禁用虚拟环境的情况下运行此命令！

## 纯原生安装

请确保同时更新依赖项 - 否则可能会出现未察觉的故障。

``` bash
git pull
pip install -U -r requirements.txt
pip install -e .

# Ensure freqUI is at the latest version
freqtrade install-ui 
```

### 更新问题

更新问题通常源于缺失依赖项（未遵循上述说明）- 或来自更新后无法安装的依赖项（例如 TA-lib）。
请参考相应的安装章节（下方链接的常见问题）。