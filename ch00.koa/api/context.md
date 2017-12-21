# Context

Koa Context 將 node 的 request 和 response 對象封裝到單個對象中，為編寫 Web 應用程序和 API 提供了許多有用的方法。這些操作在 HTTP 服務器開發中頻繁使用，它們被添加到此級別而不是更高級別的框架，這將強制中間件重新實現此通用功能。

每個請求都將創建一個 Context，並在中間件中作為接收器引用，或者 ctx 標示符，如以下代碼片段所示：

```javascript
app.use(async ctx => {
  ctx; // 這是 Context
  ctx.request; // 這是 koa Request
  ctx.response; // 這是 koa Response
});
```

為方便起見許多上下文的訪問器和方法直接委託給他們的 ctx.request 或 ctx.response，不然的話它們是相同的。例如 ctx.type 和 ctx.length 委託給 response 對象，ctx.path 和 ctx.method 委託給 request。

## API

Context 具體方法和訪問器

### ctx.req

Node 的 request 對象

### ctx.res

Node 的 response 對象

繞過 Koa 的 response 處理是不被支持的，應避免使用以下 node 屬性：

- `res.statusCode`
- `res.writeHead()`
- `res.write()`
- `res.end()`

### ctx.request

Koa 的 request 對象

### ctx.response

Koa 的 response 對象

### ctx.state

推薦的命名空間，用於透過中間件傳遞信息和你的前端視圖

```javascript
ctx.state.user = await User.find(id);
```

### ctx.app

應用程序實例引用

### ctx.cookies.get(name, [options])

透過 options 獲取 cookie `name`：

- `signed` 所請求的 cookie 應該被簽名

koa 使用 cookies 模組，其中只需傳遞參數

### ctx.cookies.set(name, value, [options])

透過 options 設置 cookie `name` 的 `value`：

- `maxAge`：一個數字表示從 Date.now() 得到的毫秒數
- `signed`：cookie 簽名值
- `expires`：cookie 過期的 Date
- `path`：cookie 路徑，默認是 '/'
- `domain`：cookie 域名
- `secure`：安全 cookie
- `httpOnly`：服務器可訪問 cookie，默認是 true
- `overwrite`：一個布林值，表示是否覆蓋以前設置的同名 cookie (默認是 false)。如果是 true，在同一請求中設置相同名稱的所有 Cookie (不管路徑或域) 是否在設置此 Cookie 時從 Set-Cookie 標投中過濾掉

### ctx.throw([status], [msg], [properties])

Helper 方法拋出一個 `.status` 屬性默認為 500 的錯誤，這將允許 Koa 做出適當地響應。

允許以下組合：

```javascript
ctx.throw(400);
ctx.throw(400, 'name required');
ctx.throw(400, 'name required', { user: user });
```

`ctx.throw(400, 'name required');` 等效於：

```javascript
const err = new Error('name required');
err.status = 400;
err.expose = true;
throw err;
```

請注意，這些是用戶級錯誤，並用 `err.expose` 標記，這意味著消息適用於客戶端響應，這通常不是錯誤消息的內容，因為您不想洩漏故障詳細信息。

你可根據需要將 properties 對象傳遞到錯誤中，對於裝載上傳給請求者的機器有好的錯誤是有用的。這用於修飾奇人機友好型錯誤並向上游請求者報告非常有用。

```javascript
ctx.throw(401, 'access_denied', { user: user });
```

koa 使用 http-errors 來創建錯誤

### ctx.assert(value, [status], [msg], [properties])

當 !value 時，Helper 方法拋出類似於 .throw() 的錯誤。這與 node 的 assert() 方法類似

```javascript
ctx.assert(ctx.state.user, 401, 'User not found. Please login!');
```

koa 使用 http-assert 作為斷言。

### cts.respond

為繞過 Koa 的內置 response 處理，你可以顯式設置 ctx.response = false。如果你想要寫入原始的 res 對象而不是讓 Koa 處理你的 response，請使用此參數。

請注意，Koa 不支持使用此功能。這可能會破壞 Koa 中間件和 Koa 本身的預期功能。使用這個屬性被認為是一個 hack，只是便於那些希望在 Koa 中使用傳統的 fn(req, res) 功能和中間件的人

## Request 別名

以下訪問器和 Request 別名等效：

- ctx.header
- ctx.headers
- ctx.method
- ctx.method=
- ctx.url
- ctx.url=
- ctx.originalUrl
- ctx.origin
- ctx.href
- ctx.path
- ctx.path=
- ctx.query
- ctx.query=
- ctx.querystring
- ctx.querystring=
- ctx.host
- ctx.hostname
- ctx.fresh
- ctx.stale
- ctx.socket
- ctx.protocol
- ctx.secure
- ctx.ip
- ctx.subdomains
- ctx.is()
- ctx.accepts()
- ctx.acceptsEncodings()
- ctx.acceptsCharsets()
- ctx.acceptsLanguages()
- ctx.get()

## Response 別名

以下訪問器和 Response 別名等效：

- ctx.body
- ctx.body=
- ctx.status
- ctx.status=
- ctx.message
- ctx.message=
- ctx.length
- ctx.length=
- ctx.type
- ctx.type=
- ctx.headerSend
- ctx.redirect()
- ctx.attachment()
- ctx.set()
- ctx.append()
- ctx.remove()
- ctx.lastModified=
- ctx.etag=
