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

Koa 應用程序是一個**包含一組中間件函數的對象**，它是按照類似堆棧的方式組織和執行的。Koa 類似於你可能遇到過的許多其他中間件系統，例如 Ruby 的 Rack，Connect 等，然而，一個關鍵的設計點是在其低級中間件層中提供高級“語法糖”。這提高了互操作性，穩定性，並使書寫中間件更加愉快。

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

Koa 應用程序不是 HTTP 服務器的 1 對 1 展現