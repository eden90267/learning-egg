# Config 配置

框架提供了強大且可擴展的配置功能，可以自動合併應用、插件、框架的配置，按順序覆蓋，且可以根據環境維護不同的配置。合併後的配置可直接從 `app.config` 獲取。

配置的管理有多種方案，以下列一些常見的方案

1. 使用平台管理配置，應用構建時將當前環境的配置放入包內，啟動時指定該配置。但應用就無法一次構建多次部署，而且本地開發環境想使用配置會變得很麻煩。
2. 使用平台管理配置，在啟動時將當前環境的配置透過**環境變量**傳入，這是比較優雅的方式，但框架對運維的要求會比較高，需要部署平台支持，同時開發環境也有相同痛點。
3. 使用代碼管理配置，在代碼中添加多個環境的配置，在啟動時傳入當前環境的**參數**即可。但無法全局配置，必須修改代碼。

我們選擇了最後一種配置方案，**配置即代碼**，配置的變更也應該經過 review 後才能發佈。應用包本身是可以部署在多個環境的，只需要指定運行環境即可。

## 多環境配置

框架支持根據環境來加載配置，定義多個環境的配置文件，具體環境請查看[運行環境配置](../ch08.execute_env/README.md)。

```
config
|- config.default.js
|- config.test.js
|- config.prod.js
|- config.unittest.js
`- config.local.js
```

`config.default.js` 為默認的配置文件，所有環境都會加載這個配置文件，一般也會作為開發環境的默認配置文件。

當指定 env 時會同時加載對應的配置文件，並覆蓋默認配置文件的同名配置。

## 配置寫法

配置文件返回的是一個 object 對象，可以覆蓋框架的一些配置，應用也可以將自己業務的配置放到這裡方便管理。

```javascript
// 配置 logger 文件的目錄，logger 默認配置由框架提供
module.exports = {
  logger: {
    dir: '/home/admin/logs/demoapp',
  }
}
```

配置文件也可以簡化的寫成 `exports.key = value` 形式

```javascript
exports.keys = 'my-cookie-secret-key';
exports.logger = {
  level: 'DEBUG'
};
```

配置文件也可以返回一個 function，可以接受 appInfo 參數：

```javascript
// 將 logger 目錄放到代碼目錄下
const path = require('path');
module.exports = appInfo => {
  return {
    logger: {
      dir: path.join(appInfo.baseDir, 'logs'),
    }
  };
}
```

內置的 appInfo 有

| appInfo | 說明                                                            |
|:--------|:---------------------------------------------------------------|
| pkg     | package.json                                                   |
| name    | 應用名，同 pkg.name                                             |
| baseDir | 應用代碼的目錄                                                   |
| HOME    | 用戶目錄，如 admin 帳戶為 /home/admin                             |
| root    | 應用根目錄，只有在 local 和 unittest 環境下為baseDir，其他都為 HOME |

`appInfo.root` 是一個優雅的適配，比如在服務器環境我們會使用 `/home/admin/logs` 作為日誌目錄，而本地開發時又不想污染用戶目錄，這樣的適配就很好解決這個問題。

## 配置加載順序

應用、插件、框架都可以定義這些配置，而且目錄結構都是一致的，但存在優先級 (應用 > 框架 > 插件)，相對於此運行環境的優先集會更高。

比如在 prod 環境加載一個配置加載順序如下：

```
-> 插件 config.default.js
-> 框架 config.default.js
-> 應用 config.default.js
-> 插件 config.prod.js
-> 框架 config.prod.js
-> 應用 config.prod.js
```

注意，插件之間也會有加載順序，但大致順序類似，具體邏輯可參考[查看加載器](https://eggjs.org/zh-cn/advanced/loader.html)。

## 合併規則

配置的合併使用 [extend2](https://github.com/eggjs/extend2) 模組進行深度拷貝，extend2 fork 字 extend，處理數組時會存在差異。

```javascript
const a = {
  arr: [ 1, 2 ],
};
const b = {
  arr: [ 3 ],
};
extend(true, a, b);
// => { arr: [ 3 ] }
```

根據上面例子，框架直接覆蓋數組而不是進行合併。

## 配置結果

框架在啟動時會把合併後的最終配置 dump 到 `run/application_config.json` (worker 進程) 和 `run/agent_config.json` (agent 進程) 中，可以用來分析問題。

配置文件中會隱藏一些字段，主要包括兩類：

- 如密碼、密鑰等安全字段，這裡可以透過 `config.dump.ignore` 配置，必須是 Set 類型，查看[默認配置](https://github.com/eggjs/egg/blob/master/config/config.default.js)
- 如函數、Buffer 等類型，JSON.stringify 後的內容特別大

還會生成 `run/application_config_meta.json` (worker 進程) 和 `run/agent_config_meta.json` (agent 進程) 文件，用來排查屬性的來源，如：

```json
{
  "logger": {
    "dir": "/path/to/config/config.default.js"
  }
}
```