# 本地開發

為提升研發體驗，我們提供了便捷方式在本地進行開發、調試、單元測試等。

這裡使用到 egg-bin 模組 (只在本地開發和單元測試使用)。

```shell
npm i egg-bin --save-dev
```

## 啟動應用

本地啟動應用進行開發活動，當我們修改代碼並保存，應用會自動重啟實時生效。

### 添加命令

```json
{
  "scripts": {
    "dev": "egg-bin dev"
  }
}
```

這樣就可透過 `npm run dev` 命令啟動應用。

### 環境配置

本地啟動應用以 `env:local` 啟動，讀取的配置是 `config.default.js` 和 `config.local.js` 合併的結果。

### 指定端口

本地啟動應用默認 7001，可指定其他端口：

```json
{
  "scripts": {
    "dev": "egg-bin dev --port 7001"
  }
}
```

## 單元測試

### 添加命令

```json
{
  "scripts": {
    "test": "egg-bin test"
  }
}
```

這樣就可透過 `npm test` 命令進行單元測試。

### 環境配置

測試用例執行時，應用是以 `env:unittest` 啟動的，讀取的配置也是 `config.default.js` 和 `config.unittest.js` 合併的結果。

### 運行特定用例文件

運行 `npm test` 時會自動執行 test 目錄下的以 `.test.js` 結尾的文件 (默認 [glob](https://www.npmjs.com/package/glob) 匹配規則 `test/**/*.test.js`)。

若要單獨執行：

```shell
TESTS=test/x.test.js npm test
```

支持 [glob](https://www.npmjs.com/package/glob) 規則

### 指定 reporter

Mocha 支持多種形式的 reporter，默認使用 `spec` reporter。

可手動設置 `TEST_REPORTER` 環境變數來指定 reporter，例如使用 `dot`：

```shell
TEST_REPORTER=dot npm test
```

### 指定用例超時時間

默認 30 秒，也可手動指定 (單位毫秒)：

```shell
TEST_TIMEOUT=5000 npm test
```

### 透過 argv 方式傳參

`egg-bin test` 除了環境變量方式，也支持直接傳參，支持  mocha 的所有參數，參見：[mocha usage](https://mochajs.org/#usage)

```shell
# 傳遞參數需額外加一個 `--`
npm test -- --help

npm test "test/**/test.js"

npm test -- --reporter=dot

npm test -- -t 3000 --grep="should GET"
```

## 代碼覆蓋率

egg-bin 已經內置了 [nyc](https://github.com/istanbuljs/nyc) 來支持單元測試自動生成代碼覆蓋率報告。

```json
{
  "scripts": {
    "cov": "egg-bin cov"
  }
}
```

```shell
$ egg-bin cov

  test/controller/home.test.js
    GET /
      ✓ should status 200 and get the body
    POST /post
      ✓ should status 200 and get the request body

  ...

  16 passing (1s)

=============================== Coverage summary ===============================
Statements   : 100% ( 41/41 )
Branches     : 87.5% ( 7/8 )
Functions    : 100% ( 10/10 )
Lines        : 100% ( 41/41 )
================================================================================
```

還可透過 `coverage/lcov-report/index.html` 打開完整的 HTML 覆蓋率報告。

### 環境配置

和 `test` 命令一樣，`cov` 命令執行時，應用也是以 `env:unittest` 啟動的，讀取的配置也是 `config.default.js` 和 `config.unittest.js` 合併的結果。

### 忽略指定文件

對於某些不需要跑測試覆蓋率的文件，可透過 `COV_EXCLUDES` 環境變量指定：

```shell
COV_EXCLUDES=app/plugins/c* npm run cov
# 或傳參
npm run cov -- --x=app/plugins/c*
```

## 調試

### 日誌輸出

#### 使用 logger 模組

框架內置 日誌 功能，使用 `logger.debug()` 輸出調試信息，**推薦在應用代碼使用它**。

```javascript
// controller
this.logger.debug('current user: %j', this.user);

// service
this.ctx.logger.debug('debug info from service');

// app/init.js
this.logger.debug('app.init');
```

透過 `config.logger.level` 來配置打印到文件的日誌級別，透過 `config.logger.consoleLevel` 配置打印到終端的日誌級別。

#### 使用 debug 模組

debug 模組是 Node.js 社區廣泛使用的 debug 工具，很多模組都使用它模組打印調試信息，Egg 社區也廣泛採用這一機制打印 debug 信息，**推薦在框架和插件開發使用它**。

我們可透過 `DEBUG` 環境變量選擇開啟指定的調試代碼，方便觀測執行過程。

(調試模組和日誌模組不要混淆，而且日誌模組也有很多功能，這裡所說的日誌都是調試信息。)

開啟所有模組日誌：

```shell
DEBUG=* npm run dev
```

開啟指定的模組：

```shell
DEBUG=egg* npm run dev
```

單元測試也可以用 `DEBUG=* npm test` 來查看測試用例運行的詳細日誌。

### 使用 egg-bin 調試

#### 添加命令

```json
{
  "scripts": {
    "debug": "egg-bin debug"
  }
}
```

這樣我們就可透過 `npm run debug` 命令來斷點調試應用。

`egg-bin` 會智能選擇調適協議，在 7.x 之後版本使用 [Inspector Protocol](https://chromedevtools.github.io/debugger-protocol-viewer/v8) 協議，低版本使用 [Legacy Protocol](https://github.com/buggerjs/bugger-v8-client/blob/master/PROTOCOL.md)。

同時也支持自定義調適參數：

```shell
egg-bin debug --proxy=9999 --inspect=9229 --inspect-brk
```

- `master` 調適端口為 9229 或 5858 (舊協議)
- `agent` 調適端口固定為 5800
- `worker` 調適端口為 `master` 調適端口遞增
- 開發階段 worker 在代碼修改後會熱重啟，導致調適端口會自增，故 `egg-bin` 啟動了代理服務，用戶可以直接 attach 9999 端口即可，毋須擔心重啟問題。

#### 環境配置

執行 `debug` 命令時，應用也是以 `env: local` 啟動的，讀取的配置是 `config.default.js` 和 `config.local.js` 合併的結果。

### 使用 DevTools 調試

最新的 DevTools 只支持 Inspector Protocol 協議，故需使用 Node.js 7.x+ 版本方能使用。

執行 `npm run debug` 啟動：

```shell
npm run debug

> showcase@1.0.0 debug /Users/tz/Workspaces/eggjs/test/showcase
> egg-bin debug

Debugger listening on ws://127.0.0.1:9229/f8258ca6-d5ac-467d-bbb1-03f59bcce85b
For help see https://nodejs.org/en/docs/inspector
2017-09-14 16:01:35,990 INFO 39940 [master] egg version 1.8.0
Debugger listening on ws://127.0.0.1:5800/bfe1bf6a-2be5-4568-ac7d-69935e0867fa
For help see https://nodejs.org/en/docs/inspector
2017-09-14 16:01:36,432 INFO 39940 [master] agent_worker#1:39941 started (434ms)
Debugger listening on ws://127.0.0.1:9230/2fcf4208-4571-4968-9da0-0863ab9f98ae
For help see https://nodejs.org/en/docs/inspector
9230 opened
Debug Proxy online, now you could attach to 9999 without worry about reload.
DevTools → chrome-devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9999/__ws_proxy__
```

然後選擇以下一種方式即可：

- 直接訪問控制台最後輸出的 `DevTools` 地址，該地址是代理後的 worker，毋須擔心重啟問題。
- 訪問 `chrome://inspect`，配置對應的端口，然後點擊 `Open dedicated DevTools for Node` 即可打開調試控制台。

![](https://user-images.githubusercontent.com/227713/30419047-a54ac592-9967-11e7-8a05-5dbb82088487.png)

### 使用 WebStorm 進行調試

`egg-bin` 會自動讀取 WebStorm 調試模式下設置的環境變量 `$NODE_DEBUG_OPTION`

使用 WebStorm 的 npm 調試啟動即可：

![](https://user-images.githubusercontent.com/227713/30423086-5dd32ac6-9974-11e7-840f-904e49a97694.png)

### 使用 VSCode 進行調試

![](https://user-images.githubusercontent.com/227713/30421285-dad801e6-996e-11e7-85b6-817165ab5783.png)

在 node.js 7.x 及之後的版本，配置 `.vscode/launch.json` 如下，然後 F5 一鍵啟動即可。

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Egg",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceRoot}",
      "runtimeExecutable": "npm",
      "windows": { "runtimeExecutable": "npm.cmd" },
      "runtimeArgs": [ "run", "debug" ],
      "console": "integratedTerminal",
      "protocol": "auto",
      "restart": true,
      "port": 9999
    }
  ]
}
```

在低於 Node.js 7.x 版本中，只能在 Terminal 手動執行 `npm run debug`，然後透過以下配置進行 attach：

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach Egg Worker",
      "type": "node",
      "request": "attach",
      "restart": true,
      "port": 9999
    }
  ]
}
```

> 後續我們會提供一個 [vscode-eggjs](https://github.com/eggjs/vscode-eggjs) 來簡化配置。

更多 VSCode Debug 可參見：[https://code.visualstudio.com/docs/nodejs/nodejs-debugging](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)

## 更多

想了解更多，或為你團隊訂製一個本地開發工具，請參考 [egg-bin](https://github.com/eggjs/egg-bin)。