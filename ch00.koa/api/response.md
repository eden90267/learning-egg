# Response

Koa Response 對象是在 node 的 vanilla 響應對象上的抽象，提供諸多對 HTTP 服務器開發有用的功能。

## API

### response.header

### response.headers

別名是 response.header

### response.socket

### response.status(=)

獲取響應狀態。默認情況下，response.status 設置為 404，而不是像 node 的 res.statusCode 那樣默認為 200。

透過數字代碼設置響應狀態：

  - 100 "continue"
  - 101 "switching protocols"
  - 102 "processing"
  - 200 "ok"
  - 201 "created"
  - 202 "accepted"
  - 203 "non-authoritative information"
  - 204 "no content"
  - 205 "reset content"
  - 206 "partial content"
  - 207 "multi-status"
  - 208 "already reported"
  - 226 "im used"
  - 300 "multiple choices"
  - 301 "moved permanently"
  - 302 "found"
  - 303 "see other"
  - 304 "not modified"
  - 305 "use proxy"
  - 307 "temporary redirect"
  - 308 "permanent redirect"
  - 400 "bad request"
  - 401 "unauthorized"
  - 402 "payment required"
  - 403 "forbidden"
  - 404 "not found"
  - 405 "method not allowed"
  - 406 "not acceptable"
  - 407 "proxy authentication required"
  - 408 "request timeout"
  - 409 "conflict"
  - 410 "gone"
  - 411 "length required"
  - 412 "precondition failed"
  - 413 "payload too large"
  - 414 "uri too long"
  - 415 "unsupported media type"
  - 416 "range not satisfiable"
  - 417 "expectation failed"
  - 418 "I'm a teapot"
  - 422 "unprocessable entity"
  - 423 "locked"
  - 424 "failed dependency"
  - 426 "upgrade required"
  - 428 "precondition required"
  - 429 "too many requests"
  - 431 "request header fields too large"
  - 500 "internal server error"
  - 501 "not implemented"
  - 502 "bad gateway"
  - 503 "service unavailable"
  - 504 "gateway timeout"
  - 505 "http version not supported"
  - 506 "variant also negotiates"
  - 507 "insufficient storage"
  - 508 "loop detected"
  - 510 "not extended"
  - 511 "network authentication required"

### response.message(=)

獲取響應的狀態消息。默認情況下，response.message 與 response.status 關聯

### response.length=

將響應的 Content-Length 設置為給定值

### response.length

以數字返回響應的 Content-Length，或從 ctx.body 推導出來，或者 undefined

### response.body

### response.body=

將響應體設置為以下之一：

- string 寫入
- Buffer 寫入
- Stream 管道
- Object || Array JSON-字符串化
- null 無內容響應

如果 response.status 未被設置，Koa 將會自動設置狀態為 200 或 204。

#### String

Content-Type 默認為 `text/html` 或 `text/plain`，同時默認字符集是 utf-8。Content-Length 字段也是如此。

#### Buffer

Content-Type 默認為 `application/octet-stream`，並且 Content-Length 字段也是如此。

#### Stream

Content-Type 默認為 application/octet-stream。

每當流被設置為響應主體時，.onerror 作為偵聽器自動添加到 error 事件中以捕獲任何錯誤。此外，每當請求關閉 (甚至過早) 時，流都將被銷毀。如果你不想要這兩個功能，請勿直接將流設為主體。例如，當將主體設置為代理中的 HTTP 流時，你可能不想要這麼做，因為它會破壞底層連接。

以下是流錯誤處理的示例，而不會自動破壞流：

```javascript
const PassThrough = require('stream').PassThrough;

app.use(async ctx => {
   ctx.body = someHTTPStream.on('error', ctx.onerror).pipe(PassThrough());
});
```

#### Object

Content-Type 默認為 application/json，這包括普通的對象 `{foo:'bar'}` 和數組 `['foo', 'bar']`。

### response.get(field)

不區分大小寫獲取響應標頭字段值 field

```javascript
const etag = ctx.response.get('ETag');
```

### response.set(field, value)

設置響應標頭 field 到 value：

```javascript
ctx.set('Cache-Control', 'no-cache');
```

### response.append(field, value)

用值 val 附加額外的標頭 field

```javascript
ctx.append('Link', '<http://127.0.0.1/>');
```

### response.set(fields)

用一個對象設置多個響應標頭 fields：

```javascript
ctx.set({
  'Etag': '1234',
  'Last-Modified': date
});
```

### response.remove(field)

刪除標頭 field

### response.type

獲取響應 Content-Type 不含參數 "charset"。

```javascript
const ct = ctx.type;
// => "image/png"
```

### response.type=

設置響應 Content-Type 透過 mime 字符串或文件擴展名

```javascript
ctx.type = 'text/plain; charset=utf-8';
ctx.type = 'image/png';
ctx.type = '.png';
ctx.type = 'png';
```

注意：在適當的情況下為你選擇 `charset`，比如 `response.type = 'html'` 將默認是 'utf-8'。如果你想覆蓋 charset，使用 ctx.get('Content-Type', 'text/html') 將響應字段設置為直接值。

### response.is(types...)

非常類似 `ctx.request.is()`. 檢查響應類型是否是所提供的類型之一。這對於創建操縱響應的中間件特別有用。例如，這是一個中間件，可以削減除流之外的所有 HTML 響應：

```javascript
const minify = require('html-minifier');

app.use(async (ctx, next) => {
  await next();
  
  if (!ctx.response.is('html')) return;
  
  let body = ctx.body;
  if (!body || body.pipe) return;
  
  if (Buffer.isBuffer(body)) body = body.toString();
  ctx.body = minify(body);
})
```

### response.redirect(url, [alt])

執行 [302] 重定向到 url

字符串 “back” 是特別提供 Referrer 支持的，當 Referrer 不存在時，使用 alt 或 "/"。

```javascript
ctx.redirect('back');
ctx.redirect('back', '/index.html');
ctx.redirect('/login');
ctx.redirect('http://google.com');
```

要更改 “302” 的默認狀態，只需在該調用之前或之後分配狀態。要變更主體請在此調用之後：

```javascript
ctx.status = 301;
ctx.redirect('/cart');
ctx.body = 'Redirecting to shopping cart';
```

### response.attachment([filename])

將 Content-Disposition 設置為 “附件” 以指示客戶端提示下載。(可選) 指定下載的 filename。

### response.headerSend

檢查是否已經發送了一個響應頭。用於查看客戶端是否可能會收到錯誤通知。

### response.lastModified(=)

### response.etag=

設置包含 `"` 包裏的 ETag 響應，請注意，沒有相應的 response.etag getter。

```javascript
ctx.response.etag = crypto.createHash('md5').update(ctx.body).digest('hex');
```

### response.vary(field)

在 field 上變化

### response.flushHeaders()

刷新任何設置的標頭，並開始主體