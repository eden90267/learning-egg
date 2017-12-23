# 框架內置基礎對象

本章會初步介紹一下框架中內置的一些基礎對象，包括 Koa 繼承而來的 4 個對象 (Application, Context, Request, Response) 以及框架擴展的一些對象 (Controller, Service, Helper, Config, Logger)，之後文檔閱讀中會經常遇到它們。

## Application

Application 是全局應用對象，在一個應用中，只會實例化一個，它繼承自 Koa.Application，在它上面我們可以掛載一些全局的方法和對象。我們可輕鬆地在插件或應用中擴展 Application 對象。

### 獲取方式

Application 對象幾乎可以在編寫應用時的任何一個地方獲取到，下面介紹幾個經常用到的獲取方式：

幾乎所有被框架 Loader 加載的文件 (Controller, Service, Schedule 等)，都可以 export 一個函數，這個函數會被 Loader 調用，並使用 app 作為參數。

- 啟動自定義腳本

  ```javascript
  // app.js
  module.exports = app => {
    app.cache = new Cache();
  };
  ```

- Controller 文件

  ```javascript
  // app/controller/user.js
  class UserController extends Controller {
    async fetch() {
      this.ctx.body = app.cache.get(this.ctx.query.id);
    }
  }
  ```

和 Koa 一樣，在 Context 對象上，可以透過 ctx.app 訪問到 Application 對象。以上面的 Controller 文件舉例：

```javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    this.ctx.body = this.ctx.app.cache.get(this.ctx.query.id);
  }
}
```

在繼承於 Controller, Service 基類的實例中，可以透過 this.app 訪問到 Application 對象。

```javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    this.ctx.body = this.app.cache.get(this.ctx.query.id);
  }
}
```

## Context

Context 是一個 **請求級別的對象**，繼承自 Koa.Context。在每一次收到用戶請求時，框架便會實例化一個 Context 對象，這個對象封裝了這次用戶請求的信息，並提供了許多便捷的方法來獲取請求參數或者設置響應信息。框架會將所有的 Service 掛載到 Context 實例上，一些插件也會將一些其他的方法和對象掛載到它上面 (egg.sequelize 會將所有的 model 掛載在 Context 上)。

### 獲取方式

最常見的 Context 實例獲取方式是在 Middleware, Controller 以及 Service 中。Controller 和 Service 獲取方式在上面例子已展示過 (Service 同 Controller)，在 Middleware 中獲取 Context 實例則和 Koa 框架在中間件獲取 Context 對象的方式一致。

框架的 Middleware 同時支持 Koa v1 和 Koa v2 兩種不同的中間件寫法：

```javascript
// Koa v1
function middleware(next) {
  // this is instance of Context
  console.log(this.query);
  yield next;
}

// Koa v2
async function middleware(ctx, next) {
  // ctx is instance of Context
  console.log(ctx.query);
}
```

除了在請求可以獲取 Context 實例之外，在有些非用戶請求的場景下我們需要訪問 service / model 等 Context 實例上的對象，我們可透過 `Application.createAnonymousContext()` 方法創建一個匿名的 Context 實例：

```javascript
// app.js
module.exports = app => {
  app.beforeStart(async () => {
    const ctx = app.createAnonymousContext();
    // preload before app start
    await ctx.service.posts.load();
  });
}
```

在定時任務中的每一個 task 都接受一個 Context 實例作為參數，以便我們更方便的執行一些定時的業務邏輯：

```javascript
// app/schedule/refresh.js
exports.task = async ctx => {
  await ctx.service.posts.refresh();
}
```

## Request & Response

Request 是一個請求級別的對象，繼承自 Koa.Request。封裝了 Node.js 原生的 HTTP Request 對象，提供了一系列輔助方法獲取 HTTP 請求常用參數。

Response 是一個請求級別的對象，繼承自 Koa.Response。封裝了 Node.js 原生的 HTTP Response 對象，提供了一系列輔助方法設置 HTTP 響應。

### 獲取方式

可以在 Context 的實例上獲取到當前請求的 Request(ctx.request) 和 Response(ctx.response) 實例。

```javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const {app, ctx} = this;
    const id = ctx.request.query.id;
    ctx.response.body = app.cache.get(id);
  }
}
```

- Koa 會在 Context 上代理一部分 Request 和 Response 上的方法和屬性，參見 [Koa.Context](http://koajs.com/#context)。
- `ctx.request.query.id` 和 `ctx.query.id` 是等價的，`ctx.response.body` 和 `ctx.body` 是等價的。
- 需要注意的是，獲取 POST 的 body 應該使用 `ctx.request.body`，而不是 `ctx.body`。

## Controller

框架提供了一個 Controller 基類，並推薦所有的 Controller 都繼承於該基類實現。這個 Controller 基類有下列屬性：

- ctx - 當前請求的 Context 實例
- app - 應用的 Application 實例
- config - 應用的配置
- service - 應用所有的 service
- logger - 為當前 controller 封裝的 logger 對象

在 Controller 文件中，可以透過兩種方式來引用 Controller 基類：

```javascript
// app/controller/user.js

// 從 egg 上獲取 (推薦)
const Controller = require('egg').Controller;
class UserController extends Controller {
  // implement
}
module.exports = UserController;

// 從 app 實例上獲取
module.exports = app => {
  return class UserController extends app.Controller {
    // implement
  }
}
```

## Service

框架提供了一個 Service 基類，並推薦所有的 Service 都繼承於該基類實現。

Service 基類的屬性和 Controller 基類屬性一致，訪問方式也類似：

```javascript
// app/service/user.js

// 從 egg 上獲取 (推薦)
const Service = require('egg').Service;
class UserService extends Service {
  // implement
}
module.exports = UserService;

// 從 app 實例上獲取
module.exports = app => {
  return class UserService extends app.Service {
    // implement
  }
}
```

## Helper

Helper 用來提供一些實用的 utility 函數。它的作用在於我們可以將一些常用的動作抽離在 helper.js 裡面成立一個獨立的函數，這樣可以用 JavaScript 來寫複雜的邏輯，避免邏輯分散各處，同時可以更好的編寫測試用例。

Helper 自身是一個類，有和 Controller 基類一樣的屬性，它也會在每次請求時進行實例化，因此 Helper 上的所有函數也能獲取到當前請求相關的上下文信息。

### 獲取方式

可以在 Context 的實例上或去到當前請求的 Helper (`ctx.helper`) 實例。

```javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const {app, ctx} = this;
    const id = ctx.query.id;
    const user = app.cache.get(id);
    ctx.body = ctx.helper.formatUser(user);
  }
}
```

除此之外，Helper 的實例還可以在模板中獲取到，例如可以在模板中獲取到 security 插件提供的 shtml 方法。

```html
<!-- app/view/home.nj -->
{{ helper.shtml(value) }}
```

### 自定義 helper 方法

應用開發中，我們可能經常要自定義一些 helper 方法，例如上面例子中的 formatUser，我們可以透過框架擴展的形式來定義 helper 方法。

```javascript
// app/extend/helper.js
module.exports = {
  formatUser(user) {
    return only(user, ['name', 'phone']);
  }
}
```

## Config

我們推薦應用開發遵循配置和代碼分離原則，將一些需要硬編碼的業務配置都放到配置文件中，同時配置文件支持各個不同的環境使用不同的配置，使用起來也非常方便，所有框架、插件和應用級別的配置都可以透過 Config 對象獲取到，關於框架配置，可詳細閱讀 Config 配置 章節。

### 獲取方式

我們可以透過 app.config 從 Application 實例上獲取到 config 對象，也可在 Controller, Service, Helper 的實例上透過 `this.config` 獲取到 config 對象。

## Logger

框架內置了功能強大的 日誌功能，可以非常方便的打印各種級別的日誌到對應的日誌文件中，每一個 logger 對象都提供了 5 個級別的方法：

- `logger.debug()`
- `logger.info()`
- `logger.warn()`
- `logger.error()`

在框架中提供了多個 Logger 對象，下面我們簡單介紹一下各個 Logger 對象的獲取方式和使用場景：

### App Logger

我們可以透過 `app.logger` 來獲取到它，如果我們想做一些應用級別的日誌紀錄，如日誌啟動階段的一些資料信息，記錄一些業務上與請求無關的信息，都可透過 App Logger 來完成。

### App CoreLogger

我們可以透過 `app.coreLogger` 來獲取到它，一般我們在開發應用時都不應該透過 CoreLogger 打印日誌，而框架和插件則需要透過它來打印應用級別的日誌，這樣可以更清晰的區分應用和框架打印的日誌，透過 CoreLogger 打印的日誌會放到和 Logger 不同的文件中。

### Context Logger

我們可透過 `ctx.logger` 從 Context 實例上獲取到它，從訪問方式上我們可以看出來，Context Logger 一定是與請求相關的，它打印的日誌都會在前面帶上一些當前請相關的信息 (如 `[$userId/$ip/$traceId/${cost}ms $method $url]`)，透過這些信息，我們可以從日誌快速定位請求，並串聯一次請求中的所有的日誌。

### Context CoreLogger

我們可以透過 ctx.coreLogger 獲取到它，和 Context Logger 的區別是一般只有插件和框架會透過它來記錄日誌。

### Controller Logger & Service Logger

我們可以在 Controller 和 Service 實例上透過 `this.logger` 獲取到它們，它們本質上就是一個 Context Logger，不過在打印日誌的時候還會額外的加上文件路徑，方便定位日誌的打印位置。

## Subscription

訂閱模型是一種比較常見的開發模式，譬如消息中間件的消費者或調度任務。因此我們提供了 Subscription 基類來規範化這種模式。

可以透過以下方式來引用 Subscription 基類：

```javascript
const Subscription = require('egg').Subscription;

class Schedule extends Subscription {
  // 需要實現此方法
  // subscribe 可以為 async function 或 generator function
  async subscribe() {
    
  }
}
```

插件開發者可以根據自己的需求基於它訂製訂閱規範，如定時任務就是使用這種規範實現的。