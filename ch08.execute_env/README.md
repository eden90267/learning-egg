# 運行環境

一個 Web 應用本身應該是無狀態的，並擁有根據運行環境設置自身能力。

## 指定運行環境

框架有兩種方式指定運行環境

1. 透過 `config/env` 文件指定，該文件的內容就是運行環境，如 prod。一般透過構建工具來生成這個文件。
2. 透過 `EGG_SERVER_ENV` 環境變量指定

其中，方式 2 比較常用，因為透過 `EGG_SERVER_ENV` 環境變量指定運行環境更加方便，比如在生產環境啟動應用：

```shell
EGG_SERVER_ENV=prod npm start
```

## 應用內獲取運行環境

框架提供變量 `app.config.env` 來表示應用當前的運行環境。

### 運行環境相關配置

不同的運行環境會對應不同的配置，具體請閱讀 [Config 配置](https://eggjs.org/zh-cn/basics/config.html)

### 與 NODE_ENV 的區別

很多 Node.js 應用會使用 `NODE_ENV` 來區分運行環境，但 `EGG_SERVER_ENV` 區分的更加精細。一般的項目開發流程包括本地開發環境、測試環境、生產環境等，除了本地開發環境和測試環境外，其他環境可統稱為**服務器環境**，服務器環境的 `NODE_ENV` 應該為 production。而且 npm 也會使用這個變量，在應用部署的時候一般不會安裝 devDependencies，所以這個值也應該為 production。

框架默認支持的運行環境及映射關係 (如果未指定 `EGG_SERVER_ENV` 會根據 `NODE_ENV` 來匹配)

| NODE_ENV   | EGG_SERVER_ENV | 說明        |
|:-----------|:---------------|:-----------|
|            | local          | 本地開發環境 |
| test       | unittest       | 單元測試    |
| production | prod           | 生產環境    |

例如，當 NODE_ENV 為 production 而 EGG_SERVER_ENV 未指定時，框架會將 EGG_SERVER_ENV 設置成 prod。

## 自定義環境

常規開發流程可能不僅僅只有以上幾種環境，Egg 支持自定義環境來適應自己的開發流程。

比如，要為開發流程集成測試環境 SIT。將 `EGG_SERVER_ENV` 設置成 `sit` (並建議設置 `NODE_ENV = production`)，啟動時會加載 `config/config.sit.js`，運行環境變數 `app.config.env` 會被設置成 sit。

## 與 Koa 的區別

在 Koa 中我們透過 `app.env` 來進行環境判斷，`app.env` 默認的值是 `process.env.NODE_ENV`。但是在 Egg (和基於 Egg 的框架) 中，配置統一都放置在 `app.config` 上，所以我們需要透過 `app.config.env` 來區分環境，`app.env` 不再使用。