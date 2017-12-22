# Koa

用 node.js 來實現 HTTP 的中間件框架，讓 Web 應用程序和 API 可以更加愉快地編寫。Koa 的中間件堆棧以類似堆棧的方式流動，允許您執行下游操作，然後過濾並操縱上游的響應。

幾乎所有 HTTP 服務器通用的方法都被直接集成到 Koa 大約 570 行源碼的代碼庫中。其中包括比如內容協商，規範節點不一致性，重定向等其他操作。

Koa 沒有綑綁任何中間件。

## 安裝

Koa 依賴 node v7.6.0 或 ES2015 即更高版本和 async 方法支持

```shell
npm i koa
```

## Hello koa

```javascript
const Koa = require('koa');
const app = new Koa();

app.use(ctx => {
  ctx.body = 'Hello Koa';
});

app.listen(3000);
```

## 中間件

Koa 是一個中間件框架，可以將兩種不同的功能作為中間件：

- async function
- common function

以下是每個不同功能記錄器的中間件示例：

### async function (node v7.6+)

```javascript
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
```

### Common function

```javascript
// 中間件通常有兩個參數 (ctx, next), ctx 是一個請求的上下文,
// next 是調用執行下游中間件的函數，在代碼執行完成後透過 then 方法返回一個 Promise

app.use((ctx, next) => {
  const start = Date.now();
  return next().then(() => {
    const ms = Date.now() - start;
    console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
  })
});
```

### koa v1.x 中間件簽名

中間件簽名在 v1.x 和 v2.x 之間已經被更改，舊的簽名已經被棄用

**舊的簽名中間件支持將在 v3 中刪除**

可參閱[遷移指南](https://github.com/demopark/koa-docs-Zh-CN/blob/master/migration.md)

## 上下文、請求和響應

每個中間件都接收一個 Koa 的 Context 對象，該對象封裝了一個傳入的 http 消息，並對該消息進行了相應的響應。ctx 通常用作上下文對象的參數名稱。

```javascript
app.use(async (ctx, next) => {await next();});
```

Koa 提供了一個 Request 對象作為 Context 的 request 屬性。Koa 的 Request 對象提供了用於處理 http 請求的方法，該請求委託給 node http 模組的 IncomingMessage。

這是一個檢查請求客戶端 xml 支持的示例：

```javascript
app.use(async (ctx, next) => {
  ctx.assert(ctx.request.accepts('xml'), 406);
  // 相當於：
  // if (!ctx.request.accepts('xml')) ctx.throw(406);
  await next();
})
```

Koa 提供了一個 Response 對象作為 Context 的 response 屬性。Koa 的 Response 對象提供了用於處理 http 響應的方法，該響應委託給 ServerResponse。

Koa 對 Node 的請求和響應對象進行委託而不是擴展它們。這種模式提供了更清晰的接口，並減少了不同中間件與 Node 本身之間的衝突，並為流處理提供了更好的支持。IncomingMessage 仍然可以直接被訪問，因為 Context 和 ServerResponse 上的 req 屬性可以直接作為 Context 上的 res 屬性訪問。

這裡是一個使用 Koa 的 Response 對象將文件作為響應流式傳輸的示例：

```javascript
app.use(async (ctx, next) => {
  await next();
  ctx.response.type = 'xml';
  ctx.response.body = fs.createReadStream('really_large.xml');
});
```

Context 對象還提供了其 request 和 response 方法的快捷方式。在前面例子中，可以使用 ctx.type 而不是 ctx.request.type，而 ctx.accepts 可以用來代替 ctx.request.accepts。

可參閱 [Request](api/request.md)、[Response](api/response.md) and [Context](api/context.md) 詳細 API 參考。

## Koa 應用程序

在執行 new Koa() 時創建的對象被稱為 Koa 應用程序對象。

應用對象是 Koa 與 node 的 http 服務器和處理中間件註冊的接口，從 http 發送到中間件，默認錯誤處理，以及上下文，請求和響應對象的設置。

了解有關應用程序對象的更多信息： [應用 API 參考](api/index.md)

## 文檔

- [使用指南](guide.md)
- [錯誤處理](error-handling.md)
- Koa 與 Express
- 常見問題
- [Koa v1.x -> v2.x](https://github.com/demopark/koa-docs-Zh-CN/blob/master/migration.md)
- API 文檔
  - Context
  - [Request](api/request.md)
  - Response
- [Koa 中間件列表](https://github.com/koajs/koa/wiki)

## Babel 配置

如果您正使用的不是 node v7.6+，我們推薦你使用 babel-preset-env 配置 babel：

```shell
npm i babel-register babel-preset-env --save
```

在你入口文件配置 babel-register：

```javascript
require('babel-register');
```

還有你的 .babelrc 配置：

```
{
  "presets": [
    ["env", {
      "targets": {
        "node": true
      }
    }]
  ]
}
```

## 故障排除

[調試 Koa](https://github.com/demopark/koa-docs-Zh-CN/blob/master/guide.md#debugging-koa)

## 運行測試

```shell
npm test
```