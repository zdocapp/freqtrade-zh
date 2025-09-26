## CORS

本节内容仅在跨源场景下需要（例如当您有多个运行在 `localhost:8081`、`localhost:8082` 等端口的机器人 API，并希望将它们整合到一个 FreqUI 实例中时）。

??? info "技术说明"
    所有基于网页的前端都受 [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)（跨源资源共享）约束。
    由于大部分对 Freqtrade API 的请求都需要身份验证，正确的 CORS 策略是避免安全问题的关键。
    同时，标准规定不允许对带凭证的请求使用 `*` 通配符 CORS 策略，因此必须适当配置此设置。

用户可通过 `CORS_origins` 配置设置允许不同源 URL 访问机器人 API。
该设置包含允许从机器人 API 消费资源的 URL 列表。

假设您的应用部署在 `https://frequi.freqtrade.io/home/` - 这意味着需要以下配置：

```jsonc
{
    //...
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": ["https://frequi.freqtrade.io"],
    //...
}
```

在以下（较常见）情况下，FreqUI 可通过 `http://localhost:8080/trade` 访问（这是在导航栏中访问 freqUI 时显示的地址）。
![freqUI url](assets/frequi_url.png)

此场景的正确配置是 `http://localhost:8080` - 即 URL 的主要部分（包含端口号）。

```jsonc
{
    //...
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": ["http://localhost:8080"],
    //...
}
```

!!! Tip "trailing Slash"
    在 `CORS_origins` 配置中不允许使用尾部斜杠（例如 `"http://localhots:8080/"`）。
    此类配置将不会生效，并且 CORS 错误仍将存在。

!!! Note
    我们强烈建议同时将 `jwt_secret_key` 设置为一个随机且仅您自己知晓的值，以避免未经授权访问您的机器人。