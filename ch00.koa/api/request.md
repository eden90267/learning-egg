# Reqeust

Koa Request 對象是在 node 的 vanilla 請求對象之上的抽象，提供了諸多對 HTTP 服務器開發有用的功能。

## API

### request.header(=)

### request.headers(=)

別名為 request.header(=)

### request.method(=)

### request.length

返回以數字返回請求的 Content-Length，或 undefined

### request.url(=)

### request.href

獲取完整的請求 URL，包括 protocol，host 和 url

```javascript
ctx.request.href;
// => http://example.com/foo/bar?q=1
```

### request.path(=)

### request.querystring(=)

### request.search

使用 ? 獲取原始查詢字符串

### request.host

獲取當前主機 (hostname:post)

### request.hostname

存在時獲取主機名

### request.URL

獲取 WHATWG 解析的 URL 對象

### request.type

獲取請求 Content-Type 不含參數 "charset"

```javascript
const ct = ctx.request.type; // => "image/png"
```

### request.charset

在存在時獲取請求字符集，或者 undefined

```javascript
ctx.request.charset; // => "utf-8"
```

### request.query(=)

獲取解析的查詢串，當沒有查詢字符串時，返回一個空對象。請注意，此 getter 不支持嵌套解析。

例如："color=blue&size=small"

```javascript
{
  color: 'blue',
  size: 'small'
}
```

### request.fetch

檢查請求緩存是否”新鮮“，也就是內容沒有改變。此方法用於 `If-None-Match / ETag`, 和 `If-Modified-Since` 和 `Last-Modified`之間的緩存協商。在設置一個或多個這些響應頭後應該引用它。

```javascript
// 新鮮度檢查需要狀態 20x 或 304
ctx.status = 200;
ctx.set('ETag', '123');

// 緩存是好的
if (ctx.fresh) {
  ctx.status = 304;
  return
}

// 緩存是陳舊的
// 獲取新數據
ctx.body = await db.find('something');
```

### request.stale

相反於 request.fresh

### request.protocol

### request.secure

透過 ctx.protocol == 'https' 來檢查請求是否透過 TLS 發出

### request.ip

請求遠程地址

### request.ips

當 `X-Forwarded-For` 存在並且 app.proxy 被啟用時，這些 ips 的數組被返回，從上游 -> 下游排序。禁用時返回一個空數組。

### request.subdomains

將子域返回為數組。

子域是應用程序主域之前主機的點分隔部分。默認情況下，應用程序的域名假定為主機的最後兩個部分。這可以透過設置 `app.subdomainOffset` 來更改。

例如，如果域名為 "tobi.ferrets.example.com"：

`app.subdomainOffset` 未設置，`ctx.subdomains` 是 `["ferrets", "tobi"]`。如果 `app.subdomainOffset` 是 3，`ctx.subdomains` 是 `["tobi"]`。

### request.is(types...)

檢查傳入請求是否包含 Content-Type 頭字段，並且包含任意的 mime type。如果沒有請求主體，返回 null。如果沒有內容類型，或匹配失敗，則返回 false，反之則返回匹配的 content-type。

```javascript
// 使用 Content-Type: text/html; charset=utf-8
ctx.is('html'); // => 'html'
ctx.is('text/html'); // => 'text/html'
ctx.is('text/*', 'text/html'); // => 'text/html'

// 當 Content-Type 是 application/json 時
ctx.is('json', 'urlencoded'); // => 'json'
ctx.is('application/json'); // => 'application/json'
ctx.is('html', 'application/*'); // => 'application/json'

ctx.is('html'); // => false
```

例如，如果要確保僅將圖像發送到給定路由：

```javascript
if (ctx.is('image/*')) {
  // 處理
} else {
  ctx.throw(415, 'images only!');
}
```

### 內容協商

Koa 的 request 對象包括由 accepts 和 negotiator 提供的有用的內容協商實體。

這些實用程序是：

- request.accepts(types)
- request.acceptsEncoding(types)
- request.acceptsCharsets(charsets)
- request.acceptsLanguages(langs)

如果沒有提供類型，則返回 **所有** 可接受的類型。

如果提供多種類型，將返回最佳匹配。如果沒有找到匹配項，則返回一個 false，你應該向客戶端發送一個 406 "Not Acceptable" 響應。

如果接收到任何類型的接收頭，則會返回第一個類型。因此，你提供的類型順序很重要。

### request.accepts(types)

檢查給定的 type(s) 是否可接受，如果 true，返回最佳匹配，否則為 false。type 值可能是一個或多個 mime 類型的字符串，如 application/json，擴展名稱如 json，或數組 `["json", "html", "text/plain"]`。

```javascript
// Accept: text/html
ctx.accepts('html');
// => "html"

// Accept: text/*, application/json
ctx.accepts('html');
// => "html"
ctx.accepts('text/html');
// => "text/html"
ctx.accepts('json', 'text');
// => "json"
ctx.accepts('application/json');
// => "application/json"

// Accept: text/*, application/json
ctx.accepts('image/png');
ctx.accepts('png');
// => false

// Accept: text/*;q=.5, application/json
ctx.accepts(['html', 'json']);
ctx.accepts('html', 'json');
// => "json"

// No Accept header
ctx.accepts('html', 'json');
// => "html"
ctx.accepts('json', 'html');
// => "json"
```

你可根據需要多次調用 ctx.accepts()，或使用 switch：

```javascript
switch (ctx.accepts('json', 'html', 'text')) {
  case 'json': break;
  case 'html': break;
  case 'text': break;
  default: ctx.throw(406, 'json, html, or text only');
}
```

### request.acceptsEncodings(encodings)

檢查 encodings 是否可接受，返回最佳匹配為 true，否則為 false。請注意，您應該將 identity 作為編碼之一！

```javascript
// Accept-Encoding: gzip
ctx.acceptsEncodings('gzip', 'deflate', 'identity');
// => "gzip"

ctx.acceptsEncodings(['gzip', 'deflate', 'identity']);
// => "gzip"
```

當沒給出參數，所有接受的編碼將作為數組返回：

```javascript
// Accept-Encoding: gzip, deflate
ctx.acceptsEncodings();
// => ["gzip", "deflate", "identity"]
```

請注意，如果客戶端顯示地發送 `identity;q=0`，那麼 `identity` 編碼 (這意味著沒有編碼) 可能是不可接受的。雖然這是一個邊緣的情況，你仍然應該處理這種方法返回 false 的情況。

### request.acceptsCharsets(charsets)

檢查 charsets 是否可接受，在 true 時返回最佳匹配，否則為 false。

```javascript
// Accept-Charset: utf-8, iso-8859-1;q=0.2, utf-7;q=0.5
ctx.acceptsCharsets('utf-8', 'utf-7');
// => "utf-8"

ctx.acceptsCharsets(['utf-7', 'utf-8']);
// => "utf-8"
```

當沒給出參數被賦予所有接受的字符集將作為數組返回：

```javascript
// Accept-Charset: utf-8, iso-8859-1;q=0.2, utf-7;q=0.5
ctx.acceptsCharsets();
// => ["utf-8", "utf-7", "iso-8859-1"]
```

### request.acceptsLanguages(langs)

檢查 langs 是否可以接受，如果為 true，返回最佳匹配，否則為 false。

```javascript
// Accept-Language: en;q=0.8, es, pt
ctx.acceptsLanguages('es', 'en');
// => "es"

ctx.acceptsLanguages(['en', 'es']);
// => "es"
```

當沒有參數被賦予所有接受的語言將作為數組返回：

```javascript
// Accept-Language: en;q=0.8, es, pt
ctx.acceptsLanguages();
// => ["es", "pt", "en"]
```

### request.idempotent

檢查請求是否是幂等的

### request.socket

### request.get(field)

返回請求標頭。