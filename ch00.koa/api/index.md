# 安裝

Koa 依賴 node v 7.6.0 或 ES 2015 以及更高版本和 async 方法支持。

你可以使用自己喜歡的版本管理器快速安裝支持的 node 版本：

```shell
nvm install 7
npm i koa
node my-koa-app.js
```

## 使用 Babel 實現 Async 方法

要在 node < 7.6 版本的 Koa 中使用 async 方法，我們推薦使用 babel's require hook

```javascript
require('babel-register');
// 應用的其餘 require 需要被放到 hook 後面
const app = require('./app');
```

要解析和編譯 async 方法，你至少應該有 `transform-async-to-generator` 或 `transform-async-to-module-method` 插件。

例如，在你的 .babelrc 文件中，你應該有：

```json
{
  "plugins": ["transform-async-to-generator"]
}
```

你也可以用 env preset 的 target 參數 "node": "current" 替代。

# 應用程序

Koa 應用程序是一個**包含一組中間件函數的對象**，它是按照類似堆棧的方式組織和執行的。Koa 類似於你可能遇到過的許多其他中間件系統，例如 Ruby 的 Rack，Connect 等，然而，一個關鍵的設計點是在其**低級中間件層中提供高級“語法糖”**。這提高了互操作性，穩定性，並使書寫中間件更加愉快。

這包括諸如內容協商，緩存清理，代理支持和重定向等常見任務的方法。儘管提供了相當多的有用方法 Koa 仍保持了一個很小的體積，因為沒有綑綁中間件。

```javascript
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => ctx.body = 'Hello World');

app.listen(3000);
```

## 級聯

Koa 中間件以更傳統的方式級聯，您可能習慣使用類似的工具 - 之前難以讓用戶友好地使用 node 的回調。然而，使用 async 功能，我們可以實現“真實”的中間件。對比 Connect 的實現，透過一系列功能直接傳遞控制，直到一個返回，Koa 調用“下游”，然後控制流回“上游”。

下面以 “Hello World” 的響應作為示例，首先請求流透過 x-response-time 和 logging 中間件來請求何時開始，然後繼續移交給 response 中間件。當一個中間件調用 next() 則該函數暫停並將控制權傳遞給定義的下一個中間件。當在下游沒有更多的中間件執行後，堆棧將展開並且每個中間件恢復執行上游行為。

```javascript
const Koa = require('koa');
const app = new Koa();

// x-response-time

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});

// logger

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}`);
});

// response

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

## 設置

應用程序設置是 app 實例上的屬性，目前支持如下：

- `app.env` 默認是 **NODE_ENV** 或 "development"
- `app.proxy` 當真正的代理頭字段將被信任時
- `app.subdomainOffset` 對於要忽略的 `.subdomains` 偏移[2]

## app.listen(...)

Koa 應用程序不是 HTTP 服務器的 1 對 1 展現。可以將一個或多個 Koa 應用程序安裝在一起以形成具有單個 HTTP 服務器的更大應用程序。

創建並返回 HTTP 服務器，將給定的參數傳遞給 Server#listen()。這些內容都記錄在 [nodejs.org](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback)

以下是一個無作用的 Koa 應用程序被綁定到 3000 端口：

```javascript
const Koa = require('koa');
const app = new Koa();
app.listen(3000);
```

這裡的 `app.listen(...)` 方法只是以下方法的語法糖：

```javascript
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
```

這意味著您可以將同一個應用程序同時作為 HTTP 和 HTTPS 或多個地址：

```javascript
const http = require('http');
const https = require('https');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
https.createServer(app.callback()).listen(3001);
```

## app.callback()

返回適用於 `http.createServer()` 方法的回調函數來處理請求。你也可以使用此回調函數將 koa 應用程序掛載到 Connect/Express 應用程序中。

## app.use(function)

將給定的中間件方法添加到此應用程序。可參閱 [Middleware](https://github.com/koajs/koa/wiki#middleware)。

## app.keys=

設置簽名的 Cookie 密鑰。

這些被傳遞給 [KeyGrip](https://github.com/crypto-utils/keygrip)，但是你也可以傳遞你自己的 KeyGrip 實例。

例如，以下是可以接受的：

```javascript
app.keys = ['im a newer secret', 'i like turtle'];
app.keys = new KeyGrip(['im a newer secret', 'i like turtle'], 'sha256');
```

這些密鑰可以倒換，並在使用 {signed: true} 參數簽名 Cookie 時使用。

## app.context

`app.context` 是從其創建 ctx 的原型。你可透過編輯 `app.context` 為 ctx 添加其他屬性。這對於將 ctx 添加到整個應用程序中使用的屬性或方法非常有用，這可能會更加有效 (不需要中間件) 和/或 更簡單 (更少的 `require()`)，而更多地依賴於 ctx，這可以被認為是一種反模式。

例如，要從 ctx 添加對資料庫的引用：

```javascript
app.context.db = db();

app.use(async ctx => {
  console.log(ctx.db);
});
```

注意：

- ctx 上的許多屬性都是使用 `getter`，`setter` 和 `Object.defineProperty()` 定義的。你只能透過在 app.context 上使用 Object.defineProperty() 來編輯這些屬性 (不推薦)。查閱 [https://github.com/koajs/koa/issues/652](https://github.com/koajs/koa/issues/652)。
- 安裝的應用程序目前使用其父級的 ctx 和設置。因此，安裝的應用程序只是一組中間件。

## 錯誤處理

默認情況下，將所有錯誤輸出到 stderr，除非 app.silent 為 true。當 err.status 是 404 或 err.expose 是 true 時默認錯誤處理程序也不會輸出錯誤。要執行自定義錯誤處理邏輯，如集中式日誌紀錄，你可添加一個 “error” 事件監聽器：

```javascript
app.on('error', err => {
  log.error('server error', err);
});
```

如果 req/res 期間出現錯誤，並且無法響應客戶端，context 實例仍然被傳遞：

```javascript
app.on('error', (err, ctx) => {
  log.error('server error', err, ctx);
});
```

當發生錯誤 _並且_ 仍然可以響應客戶端時，也沒有數據被寫入 socket 中，Koa 將用一個 500 “內部服務器錯誤” 進行適當的響應。在任一情況下，為了記錄目的，都會發出應用級 “錯誤”。