# controller

## 什麼是 Controller

前面章節提到，我們透過 Router 將用戶的請求基於 method 和 URL 分發到對應的 Controller 上，那 Controller 負責做什麼？

簡單的說 Controller 負責**解析用戶的輸入，處理後返回相應的結果**，例如

- 在 RESTful 接口中，Controller 接受用戶的參數，從資料庫中查找內容給用戶或者將用戶的請求更新到資料庫中。
- 在 HTML 頁面請求中，Controller 根據用戶訪問不同的 URL，渲染不同的模板得到 HTML 返回給用戶。
- 在代理服務器中，Controller 將用戶的請求轉發到其他服務器上，並將其他服務器的處理結果返回給用戶。

框架推薦 Controller 層主要對用戶的請求參數進行處理 (效驗、轉換)，然後調用對應的 service 方法處理業務，得到業務結果後封裝並返回：

1. 獲取用戶透過 HTTP 傳遞過來的請求參數
2. 效驗、組裝參數
3. 調用 Service 進行業務處理，必要時處理轉換 Service 的返回結果，讓它適應用戶的需求
4. 透過 HTTP 將結果響應給用戶

## 如何編寫 Controller

所有的 Controller 文件都必須放在 `app/controller` 目錄下，可支持多及目錄，訪問的時候可透過目錄名級聯訪問。Controller 支持多種形式進行編寫，可以根據不同的項目場景和開發習慣來選擇。

### Controller 類 (推薦)

```javascript
// app/controller/post.js
const Controller = require('egg').Controller;
class PostController extends Controller {
  async create() {
    const {ctx, service} = this;
    const createRule = {
      title: {type: 'string'},
      content: {type: 'string'},
    };
    // 校驗參數
    ctx.validate(createRule);
    // 組裝參數
    const author = ctx.session.userId;
    const req = Object.assign(ctx.require.body, {author});
    // 調用 Service 進行業務處理
    const res = await service.post.create(req);
    // 設置響應內容和響應狀態碼
    ctx.body = {id: res.id};
    ctx.status = 201;
  }
}
module.exports = PostController;
```

```javascript
// app/router.js
module.exports = app => {
  const {router, controller} = app;
  router.post('createPost', '/api/posts', controller.post.create);
}
```

Controller 支持多級目錄，例如 `app/controller/sub/post.js`，則可以在 router 這樣用：

```javascript
// app/router.js
module.exports = app => {
  const {router, controller} = app;
  router.post('createPost', '/api/posts', controller.sub.post.create);
}
```

定義的 Controller 類，會在每一個請求訪問到 server 時實例化一個全新的對象，＝項目中的 Controller 類繼承於 egg.Controller，會有下面幾個屬性掛載 this 上。

- this.ctx：透過它可拿到框架封裝好的處理當前請求的各種便捷屬性和方法
- this.app：透過它可拿到框架提供的全局對象和方法
- this.service：等價於 this.ctx.service
- this.config
- this.logger：透過這個 logger 對象紀錄的日誌，在日誌前面會加上打印該日誌的文件路徑，以便快速定位日誌打印位置。

### 自定義 Controller 基類

按照類的方式編寫 Controller，不僅可讓我們更好的對 Controller 層代碼進行抽象 (例如將一些統一的處理抽象成一些私有方法)，還可透過自定義 Controller 基類的方式封裝應用中常用的方法。

```javascript
// app/core/base_controller.js
const {Controller} = require('egg');
const BaseController extends Controller {
  get user() {
    return thos.ctx.session.user;
  }
  success(data) {
    this.ctx.body = {
      success: true,
      data
    }
  }
  notFound(msg) {
    msg = msg || 'not found';
    this.ctx.throw(404, msg);
  }
}
module.exports = BaseController;
```

此時編寫應用的 Controller 時，可繼承 BaseController：

```javascript
//app/controller/post.js
const Controller = require('../core/base_controller');
class PostController extends Controller {
  async list() {
    const posts = await this.service.listByUser(this.user);
    this.success(posts);
  }
}
```

### Controller 方法 (不推薦使用，只是為了兼容)

每一個 Controller 都是一個 async function，它的入參為請求的上下文 Context 對象的實例，透過它我們可以拿到框架封裝好的各種便捷屬性和方法。

例如寫一個對應到 `POST /api/posts` 接口的 Controller：

```javascript
// app/controller/post.js
exports.create = async ctx => {
  const createRule = {
    title: { type: 'string' },
    content: { type: 'string' },
  };
  ctx.validate(createRule);
  
  const author = ctx.session.userId;
  const req = Object.assign(ctx.request.body, { author });
  
  const res = await ctx.service.post.create(req);
  
  ctx.body = { id: res.id };
  ctx.status = 201;
};
```

在上面的例子中我們引入了許多新的概念，但還是比較直觀，容易理解的，我們會在下面對它進行更詳細的介紹。

## HTTP 基礎

由於 Controller 基本上是業務開發中唯一和 HTTP 協議打交道的地方，首先來看一下 HTTP 協議是怎樣的。

如果我們發起一個 HTTP 請求來訪問前面例子中提到的Controller：

```shell
curl -X POST http://localhost:3000/api/posts --data '{"title":"controller", "content": "what is controller"}' --header 'Content-Type:application/json; charset=UTF-8'
```

透過 curl 發出的 HTTP 請求的內容就會是下面這樣的：

```shell
POST /api/posts HTTP/1.1
Host: localhost:3000
Content-Type: application/json; charset=UTF-8

{"title": "controller", "content": "what is controller"}
```

請求的第一行包含三個信息，我們比較常用前面兩個：

- method
- path

從第二行開始直到遇到第一個空行位置，都是請求的 Headers 部分，這一部份中有許多常用的屬性，包括這裡看到的 Host，Content-Type，還有 Cookie，User-Agent 等等。在這個請求中有兩個頭：

- Host：我們在瀏覽器發起請求的時候，域名會用來透過 DNS 解析找到服務的 IP 地址，但是瀏覽器也會將域名和端口號放在 Host 頭中一併發送給服務端。
- Content-Type：當我們請求有 body 的時候，都會有 Content-Type 來標明我們的請求體是什麼格式的。

之後的內容全部都是請求的 body，當請求是 POST，PUT，DELETE 等方法，可帶上請求體，服務端會根據 Content-Type 來解析請求體。

在服務端處理完這個請求後，會發送一個 HTTP 響應給客戶端

```shell
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Content-Length: 8
Date: Mon, 09 Jan 2017 08:40:28 GMT
Connection: keep-alive

{"id": 1}
```

第一行包含三段，最常用主要是響應狀態碼，這個例子中它的值是 201，它的含義是在 server 端成功創建一條資源。

第二行到下一個空行前都是響應頭。

最後剩下的部分就是這次響應真正的內容。

## 獲取 HTTP 請求參數

從上面的 HTTP 請求示例中可以看到，有好多地方可以放用戶的請求數據，框架透過在 Controller 上榜定的 Context 實例，提供了許多便捷方法和屬性獲取用戶透過 HTTP 請求發送過來的參數。

### query

常用 GET 類型的請求傳遞參數。例如 `GET /posts?category=egg&language=node` 中 `category=egg&language=node` 就是用戶傳遞過來的參數。我們可透過 ctx.query 拿到解析過後的這個參數體：

```javascript
class PostController extends Controller {
  async listPosts() {
    const query = this.ctx.query;
    // {
    //   category: 'egg',
    //   language: 'node',
    // }
  }
}
```

當 Query String 中的 key 重複，ctx.query 只取 key 第一次出現的值，後面再出現的都會被忽略。`GET /posts?category=egg&category=koa` 透過 `ctx.query` 拿到的值：`{ category: 'egg' }`

這樣處理是為了保持統一性。由於通常情況下我們都不會設計讓用戶傳遞 key 相同的 Query String，所以我們稱昂會寫類似下面代碼：

```javascript
const key = ctx.query.key || '';
if (key.startsWith('egg')) {
  // do something
}
```

而如果有人故意發起請求在 Query String 中帶上重複的 key 來請求時就會引發系統異常。因此框架保證了從 `ctx.query` 上獲取的參數一旦存在，一定是字符串類型。

### queries

有時候我們的系統會設計讓用戶傳遞相同 key，例如 GET /posts?category=egg&id=1&id=2&id=3。針對此狀況，框架提供了 ctx.queries 對象，這對象也解析了 Query String，但他不會丟棄任何一個重複的數據，而是把他們都放到數組中：

```javascript
// GET /posts?category=egg&id=1&id=2&id=3
class PostController extends Controller {
  async listPosts() {
    console.log(this.ctx.queries);
    // {
    //   category: [ 'egg' ],
    //   id: [ '1', '2', '3' ],
    // }
  }
}
```

### Router params

在 Router 中，我們介紹 Router 也可申明參數，這些參數可用 `ctx.params` 獲取到：

```javascript
// app.get('/projects/:projectId/app/:appId', 'app.listApp');
// GET /projects/1/app/2
class AppController extends Controller {
  async listApp() {
     assert.equal(this.ctx.params.projectId, '1');
     assert.equal(this.ctx.params.appId, '2');
  }
}
```

### body

雖然我們可透過 URL 傳遞參數，但還是有諸多限制：

- 瀏覽器會對 URL 的長度有所限制，如果需要傳遞的參數過多就會無法傳遞
- 服務器經常會將訪問的完整 URL 記錄到日誌文件，有一些敏感數據透過 URL 傳遞會不安全

通常會在 POST、PUT 和 DELETE 等方法的參數。一般請求中有 body 的時候，客戶端會同時發送 Content-Type 告訴服務端這次請求 body 是什麼格式。Web 開發中數據最常見的兩類格式分別是 JSON 和 Form。

框架內置 `bodyParser` 中間件來對這兩類格式的請求 body 解析成 object 掛載到 `ctx.request.body` 上。HTTP 協議中並不建議在透過 GET、HEAD 方法訪問時傳遞 BODY，所以我們無法在 GET、HEAD 方法中按照此方法獲取到內容。

```javascript
// POST /api/posts HTTP/1.1
// Host: localhost:3000
// Content-Type: application/json; charset=UTF-8
//
// {"title": "controller", "content": "what is controller"}
class PostController extends Controller {
  async listPosts() {
    assert.equal(this.ctx.request.body.title, 'controller');
    assert.equal(this.ctx.request.body.content, 'what is controller');
  }
}
```

框架對 bodyParser 設置了一些默認參數，配置好後有以下特性：

- 當請求的 Context-Type 為 `application/json`，`application/json-patch+json`，`application/vnd.api+json` 和 `application/csp-report` 時，會按照 json 格式對請求 body 進行解析，並限制 body 最大長度為 100kb。
- 當請求的 Content-Type 為 `application/x-www-form-urlencoded` 時，會按照 form 格式對請求 body 進行解析，並限制 body 最大長度為 100kb
- 如果解析成功，body 一定會是一個 Object (可能會是一個數組)

一般來說我們最經常調整的配置項就是變更解析時允許的最大長度：

```javascript
// config/config.default.js
module.exports = {
  bodyParser: {
    jsonLimit: '1mb',
    formLimit: '1mb'
  }
};
```

如果用戶請求 body 超過我們配置的解析最大長度，會拋出一個狀態碼為 413 的異常，如果用戶請求的 body 解析失敗 (錯誤的 JSON)，會拋出一個狀態碼為 400 的異常。

### 獲取上傳的文件

請求 body 除了可帶參數，還可發送文件，一般來說，瀏覽器都是透過 `Multipart/form-data` 格式發送文件的，框架透過內置 `Multipart` 插件來支持獲取用戶上傳的文件。

完整上傳示例：[https://github.com/eggjs/examples/tree/master/multipart](https://github.com/eggjs/examples/tree/master/multipart)

在 Controller 中，我們可透過 `ctx.getFileStream()` 接口能獲得到上傳的文件流。

```html
<form method="POST" action="/upload?_csrf={{ ctx.csrf | safe }}" enctype="multipart/form-data">
  title: <input name="title" />
  file: <input name="file" type="file" />
  <button type="submit">上傳</button>
</form>
```

```javascript
const path = require('path');
const sendToWormhole = require('stream-wormhole');
const Controller = require('egg').Controller;

class UploaderController extends Controller {
  async reload() {
    const ctx = this.ctx;
    const stream = await ctx.getFileStream();
    const name = 'egg-multipart-test/' + path.basename(stream.filename);
    // 文件處理，上傳到雲存儲等等
    let result;
    try {
      result = await ctx.oss.put(name, stream);
    } catch (err) {
      // 必須將上傳的文件流消費掉，要不然瀏覽器響應會卡死
      await sendToWormhole(stream);
      throw err;
    }
    
    ctx.body = {
      url: result.url,
      // 所有表單字段都能透過 `stream.fields` 獲取到
      fields: stream.fields
    }
  }
}

module.exports = UploaderController;
```

要透過 `ctx.getFileStream` 便捷的獲取到用戶上傳的文件，需要滿足兩個條件：

- 只支持上傳一個文件
- 上傳文件必須在所有其他的 fields 後面，否則在拿到文件流時可能還獲取不到 fields。

如果要獲取上傳的多個文件，不能透過 `ctx.getFileStream()` 來獲取，只能透過下面這種方式：

```javascript
const sendToWormhole = require('stream-wormhole');
const Controller = require('egg').Controller;

class UploaderController extends Controller {
  async upload() {
    const ctx = this.ctx;
    const parts = ctx.multipart();
    let part;
    // ports() return a promise
    while((part = await parts()) != null) {
      if (part.length) {
        // 如果是數組的話是 field
        console.log('field: ' + part[0]);
        console.log('value: ' + part[1]);
        console.log('valueTruncated: ' + part[2]);
        console.log('fieldnameTruncated: ' + part[3]);
      } else {
        if (!part.filename) {
          // 這時是用戶沒有選擇文件就點擊了上傳(part 是 file stream，但是 part.filename 為空)
          // 需要作出處理，例如給出錯誤提示消息
          return;
        }
        // part 是上傳的文件流
        console.log('field: ' + part.fieldname);
        console.log('filename: ' + part.filename);
        console.log('encoding: ' + part.encoding);
        console.log('mime: ' + part.mime);
        // 文件處理，上傳到雲存儲等等
        let result;
        try {
          result = await ctx.oss.put('egg-multipart-test/' + part.filename, part);
        } catch (err) {
          // 必須將上傳的文件流消費掉，要不然瀏覽器響應會卡死
          await sendToWormhole(part);
          throw err;
        }
        console.log(result);
      }
    }
    console.log('and we are done parsing the form!');
  }
}

module.exports = UploaderController;
```

為了保證文件上傳的安全，框架限制了支持的文件格式，框架默認支持白名單如下：

```json
// images
'.jpg', '.jpeg', // image/jpeg
'.png', // image/png, image/x-png
'.gif', // image/gif
'.bmp', // image/bmp
'.wbmp', // image/vnd.wap.wbmp
'.webp',
'.tif',
'.psd',
// text
'.svg',
'.js', '.jsx',
'.json',
'.css', '.less',
'.html', '.htm',
'.xml',
// tar
'.zip',
'.gz', '.tgz', '.gzip',
// video
'.mp3',
'.mp4',
'.avi',
```

用戶可透過在 `config/config.default.js` 中配置來新增支持的文件擴展名，或者重寫整個白名單

- 新增支持的文件擴展名

```javascript
module.exports = {
  multiple: {
    fileExtensions: ['.apk'],
  }
}
```

- 覆蓋整個白名單

```javascript
module.exports = {
  multiple: {
    whitelist: ['.png'],
  }
}
```

**注意，當傳遞了 whilelist 屬性，fileExtensions 屬性不生效。**

### header

除了從 URL 和請求 body 上獲取參數之外，還有許多參數是透過請求 header 傳遞的，框架提供了一些輔助屬性和方法來獲取。

- `ctx.headers`、`ctx.header`、`ctx.request.headers`、`ctx.request.header`：等價方法，都是獲取 header 對象
- `ctx.get(name)`、`ctx.request.get(name)`：獲取 header 中一個字段的值，如果字段不存在，會返回空字符串
- 建議用 `ctx.get(name)` 而不是 `ctx.headers['name']`，因為前者會自動處理大小寫

由於 header 比較特殊，有一些是 HTTP 協議規定了具體含義的 (例如 Content-Type、Accept)，有些是反向代理設置的，已經規定俗成 (X-Forwarded-For)，框架也會對他們增加一些便捷的 getter，詳細 getter 可查看 [API](https://eggjs.org/api/) 文檔。

特別是如果我們透過 `config.proxy = true` 設置了應用部署在反向代理 (Nginx) 之後，有一些 Getter 的內部處理會發生改變。

#### ctx.host

優先讀透過 `config.hostHeaders` 中配置的 header 的值，讀不到時再嘗試讀取 host 這個 header 的值，如果都讀取不到，返回空字符串。

`config.hostHeaders` 默認配置為 `x-forwarded-host`

#### ctx.protocol

透過這個 Getter 獲取 protocol 時，首先會判斷當前連接是否是加密連接，返回 https。

如果是非加密連接，優先讀透過 `config.protocolHeaders` 中配置的 header 的值來判斷是 HTTP 還是 https，如果讀取不到，我們可在配置中透過 `config.protocol` 來設置兜底值，默認為 HTTP。

`config.protocolHeaders` 默認設置為 `x-forwarded-proto`。

#### ctx.ips

透過 `ctx.ips` 獲取請求經過所有中間設備 IP 地址列表，只有在 `config.proxy = true`，才會透過讀取 `config.ipHeaders` 中配置的 header 的值來獲取，獲取不到為空數組。

`config.ipHeaders` 默認配置為 `x-forwarded-for`。

#### ctx.ip

透過 `ctx.ip` 獲取請求發起方的 IP 地址，優先從 `ctx.ips` 中獲取，`ctx.ips` 為空時使用連接上發起方的 IP 地址。

**注意：`ip` 和 `ips` 不同，`ip` 當 `config.proxy = false` 時會返回當前連接發起者的 `ip` 地址，`ips` 此時會為空數組**。

### Cookie

HTTP 請求都是無狀態的，但我們 Web 應用通常都需要知道發起請求的人是誰。為解決這個問題，HTTP 協議設計一個特殊請求頭：Cookie。服務端可透過響應頭 (set-cookies) 將少量數據響應給客戶端，瀏覽器會遵循協議將資料保存，並在下次請求同一服務的時候帶上 (瀏覽器也會遵循協議，只在訪問符合 Cookie 指定規則的網站時帶上對應的 Cookie 來保證安全性)。

透過 `ctx.cookies`，我們可以在 Controller 中便捷、安全的設置和讀取 Cookie。

```javascript
class CookieController extends Controller {
  async add() {
    const ctx = this.ctx;
    let count = ctx.cookies.get('count');
    count = count ? Number(count) : 0;
    ctx.cookies.set('count', ++count);
    ctx.body = count;
  }
  
  async remove() {
    const ctx = this.ctx;
    const count = ctx.cookies.set('count', null);
    ctx.status = 204;
  }
}
```

Cookie 雖然在 HTTP 中只是一個頭，但透過 `foo=bar;foo1=bar1;` 的格式可設置多個鍵值對。

Cookie 在 Web 應用中經常承擔傳遞客戶端身份應用信息的作用，因此有許多安全相關的設置，不可忽視，[Cookie](https://eggjs.org/zh-cn/core/cookie-and-session.html#cookie) 文檔中詳細介紹了 Cookie 的用法和安全相關的配置項，可深入閱讀了解。

### Session

透過 Cookie，我們可給每一個用戶設置一個 Session，用來存儲用戶身份相關的信息，這份信息會加密後存儲在 Cookie 中，實現跨請求的用戶身份保持。

框架內置 [Session](https://github.com/eggjs/egg-session) 插件，給我們提供 `ctx.session` 來訪問或修改當前用戶 Session。

```javascript
class PostController extends Controller {
  async fetchPosts() {
    const ctx = this.ctx;
    // 獲取 Session 上的内容
    const userId = ctx.session.userId;
    const posts = await ctx.service.post.fetch(userId);
    // 修改 Session 的值
    ctx.session.visited = ctx.session.visited ? ctx.session.visited++ : 1;
    ctx.body = {
      success: true,
      posts,
    };
  }
}
```

Session 的使用方法非常直觀，直接讀取它或修改它就可以了，如果要刪除，直接賦值為 null：

```javascript
class SessionController extends Controller {
  async deleteSession() {
    this.ctx.session = null;
  }
};
```

和 Cookie 一樣，Session 也有許多安全等選項和功能，在使用之前也最好閱讀 [Session](https://github.com/eggjs/egg-session) 文檔深入了解。

#### 配置

對於 Session 來說，主要有下面幾個屬性可在 `config.default.js` 中進行配置：

```javascript
module.exports = {
  key: 'EGG_SESS', // 乘載 Session 的 Cookie 鍵值對名字
  maxAge: 86400000, // Session 的最大有效時間
}
```