# 插件

插件機制是 Egg 框架的一大特色。它不但可保證框架核心的精簡、穩定、高效，還可促進業務邏輯的復用，生態圈的形成。

## 為什麼要插件

我們使用 Koa 中間件過程發現下面一些問題：

1. 中間件加載其實有先後順序，但中間件自身卻無法管理這種順序，只能交給使用者。一旦順序不對，結果可能有天壤之別。
2. 中間件定位是攔截用戶請求，在它前後做點事情。但實際情況是，有些功能是和請求無關的，例如：定時任務、消息訂閱、後台邏輯等等。
3. 有些功能包含非常複雜的初始化邏輯，需要在應用啟動的時候完成。這顯然也不適合放到中間件去實現。

綜上所述，我們需要一套更強大的機制，來管理、編排那些相對獨立的業務邏輯。

### 中間件、插件、應用的關係

一個插件其實就是一個「迷你的應用」，和應用幾乎一樣：

- 包含 Service、中間件、配置、框架擴展等等
- 它沒有獨立的 Router 和 Controller

他們的關係是：

- 應用可以直接引入 Koa 的中間件
- 當遇到上一節提到的場景，則應用須引入插件
- 插件本身可以包含中間件
- 多個插件可以包裝為一個上層框架

## 使用插件

插件一般透過 npm 模組的方式進行復用：

```shell
npm i egg-mysql --save
```

注意：我們建議透過 `^` 的方式引入依賴，並強烈不建議鎖定版本。

```json
{
  "dependencies": {
    "egg-mysql": "^3.0.0"
  }
}
```

然後需要在應用或框架的 `config/plugin.js` 中聲明：

```javascript
// config/plugin.js
// 使用 mysql 插件
exports.mysql = {
  enable: true,
  package: 'egg-mysql',
};
```

### 參數介紹

plugin.js 中每個配置項支持：

- `{Boolean} enable`
- `{String} package` - npm 模組名稱
- `{String} path` - 插件絕對路徑，跟 package 配置互斥
- `{Array} env` - 只有在指定運行環境才能開啟，會覆蓋插件自身 `package.json` 中的配置

### 開啟和關閉

在上層框架內部內置的插件，應用在使用時就不用配置 package 或 path，只需指定 enable 與否：

```javascript
// 對於內置插件，可用下面的簡潔方式開啟或關閉
exports.onerror = false;
````

### 根據環境配置

雖上述有 env 字段可配置，但我們更推薦 `plugin.{env}.js` 這種模式，會根據運行環境加載插件配置。

比如定義一個開發環境用的插件 `egg-dev`，只希望在本地環境加載：

```json
// npm i egg-dev --save-dev
// package.json
{
  "devDependencies": {
    "egg-dev": "*"
  }
}
```

```
// config/plugin.local.js
exports.dev = {
  enable: true,
  package: 'egg-dev',
};
```

這樣在生產環境可以 `npm i --production` 不需要下載 `egg-dev` 包了。

**注意：不存在 `plugin.default.js`。**

### package 和 path

- package 是 npm 方式引入，也是最常見的引入方式
- path 是絕對路徑引入，如應用內部抽了一個插件，但還沒到開源發布 npm 階段，或應用自己覆蓋了框架的一些插件

```javascript
// config/plugin.js
const path = require('path');
exports.mysql = {
  enable: true,
  package: path.join(__dirname, '../lib/plugin/egg-mysql'),
};

```

## 插件配置

插件一般會包含自己的默認配置，應用開發者可以在 `config.default.js` 覆蓋對應的配置：

```javascript
// config/config.default.js
exports.mysql = {
  client: {
    host: 'mysql.com',
    port: '3306',
    user: 'test_user',
    password: 'test_password',
    database: 'test',
  },
};
```

具體合併規則可參見[配置](https://eggjs.org/zh-cn/basics/config.html)。

## 插件列表

- 框架默認內置了企業級應用常用的插件：
  - onerror 統一異常處理
  - Session
  - i18n 多語言
  - watcher 文件和文件夾監控
  - multipart
  - security
  - development 開發環境配置
  - logrotator 日誌切分
  - schedule
  - static
  - jsonp
  - view
- 更多插件：[egg-plugin](https://github.com/topics/egg-plugin)

## 如何開發一個插件

[插件開發](https://eggjs.org/zh-cn/advanced/plugin.html)