# Router 路由

Router 主要用來描述請求 URL 和具體承擔執行動作的 Controller 的對應關係，框架約定了 `app/router.js` 文件用於統一所有路由規則。

透過統一配置，我們可以避免路由規則邏輯散落在多個地方，從而出現未知的衝突，集中一起可更方便查看全局路由規則。

## 如何定義 Router

- `app/router.js` 定義 URL 路由規則

```javascript
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.get('/user/:id', controller.user.info);
}
```

- `app/controller` 目錄實現 Controller

```javascript
class UserController extends Controller {
  async info() {
    this.ctx.body = {
      name: `hello ${ctx.params.id}`
    };
  }
}
```

當用戶執行 `GET /user/123`，`user.js` 這個裡面的 info 方法就會執行。

## Router 詳細定義說明

下面是路由的完整定義，參數可根據場景不同，自由選擇：

```javascript
router.verb('path-match', app.controller.action);
router.verb('router-name', 'path-match', app.controller.action);
router.verb('path-match', middleware1, ..., middlewareN, app.controller.action);
router.verb('router-name', 'path-match', middleware1, ..., middlewareN, app.controller.action);
```

路由完整定義主要包括 5 個主要部分：

- verb - 用戶觸發操作

  - router.header
  - router.options
  - router.get
  - router.post
  - router.post
  - router.path
  - router.delete
  - router.del - 由於 delete 是一個保留字，所以提供了一個 delete 方法的別名
  - router.redirect - 可對 URL 進行重定向處理

- router-name - 給路由設定一個別名，可以透過 Helper 提供的輔助函數 pathFor 和 urlFor 來生成 URL。(可選)
- path-match - 路由 URL 路徑
- middleware1 - 在 Router 裡面可以配置多個 Middleware。(可選)
- controller - 指定路由映射到具體的 controller 上，controller 可以有兩種寫法：

  - `app.controller.user.fetch` - 直接指定一個具體的 controller
  - `'user.fetch'` - 可以簡寫為字符串形式

### 注意事項

- 在 Router 定義中，可以支持多個 Middleware 串聯執行
- Controller 必須定義在 `app/controller` 目錄中
- 一個文件裡面也可以包含多個 Controller 定義，在定義路由的時候，可以透過 `${fileName}.${functionName}` 的方式制定對應的 Controller。
- Controller 支持子目錄，在定義路由的時候，可以透過 `${directoryName}.${fileName}.${functionName}` 的方式制定對應的 Controller。

```javascript
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.get('/home', controller.home);
  router.get('/user/:id', controller.user.page);
  router.post('/admin', isAdmin, controller.admin);
  router.post('/user', isLoginUser, hasAdminPermission, controller.user.create);
  router.post('/api/v1/comments', controller.v1.comments.create); // app/controller/v1/comments.js
};
```

### RESTful 風格的 URL 定義

如果想透過 RESTful 的方式定義路由，我們提供了 `app.resources('routerName', 'pathMatch', controller)` 快速在一個路徑上生成 CRUD 路由結構。

```javascript
// app/router.js
module.exports = app => {
  const {router, controller} = app;
  router.resources('posts', '/api/posts', controller.posts);
  router.resources('users', '/api/v1/users', controller.v1.users); // app/controller/v1/rusers.js
}
```

上面代碼就在 `/posts` 路徑上部署了一組 CRUD 路徑結構，對應的 Controller 為 `app/controller/posts.js`，接下來，你只需要在 `posts.js` 裡面實現對應的函數就可以了。

| Method | Path            | Route Name | Controller.Action             |
|:-------|:----------------|:-----------|:------------------------------|
| GET    | /posts          | posts      | app.controllers.posts.index   |
| GET    | /posts/new      | new_post   | app.controllers.posts.new     |
| GET    | /posts/:id      | post       | app.controllers.posts.show    |
| GET    | /posts/:id/edit | edit_post  | app.controllers.posts.edit    |
| POST   | /posts          | posts      | app.controllers.posts.create  |
| PUT    | /posts/:id      | post       | app.controllers.posts.update  |
| DELETE | /posts/:id      | post       | app.controllers.posts.destroy |

```javascript
// app/controller/posts.js
exports.index = async () => {}; // /posts

exports.new = async () => {};   // /posts/new

exports.show = async() => {};   // /posts/:id  

exports.edit = async() => {};   // /posts/:id/edit

exports.create = async() => {}; // /posts

exports.update = async() => {}; // /posts/:id

exports.destroy = async() => {};// /posts/:id
```

如果我們不需要其中的某幾種方法，可以不用在 posts.js 裡面實現，這樣對應 URL 路徑也不會註冊到 Router。

## router 實戰

下面透過更多實際的例子，來說明 router 的用法。

### 參數獲取

#### Query String 方式

```javascript
// app/router.js
module.exports = app => {
  app.router.get('/search', app.controller.search.index);
}

// app/controller/search.js
exports.index = async ctx => {
  ctx.body = `search: ${ctx.query.name}`;
};

// curl http://127.0.0.1:7001/search?name=egg
```

#### 參數命名方式

```javascript
// app/router.js
module.exports = app => {
  app.router.get('/user/:id/:name', app.controller.user.info);
};

// app/controller/user.js
exports.info = async ctx => {
  ctx.body = `user: ${ctx.params.id}, ${ctx.params.name}`;
}

// curl http://127.0.0.1:7001/user/123/xiaoming

```

#### 複雜參數的獲取

路由裡面也支持定義路由，可以更加靈活的獲取參數：

```javascript
// app/router.js
module.exports = app => {
  app.router.get(/^\/package\/([\w-.]+\/[\w-.]+)$/, app.controller.package.detail);
};

// app/controller/package.js
exports.detail = async ctx => {
  // 如果請求 URL 被正則匹配，可以按照捕獲分組的順序，從 ctx.params 中獲取
  // 按照下面的用戶請求，`ctx.params[0]` 的 內容就是 `egg/1.0.0`
  ctx.body = `package:${ctx.params[0]}`;
}
```

### 表單內容的獲取

```javascript
// app/router.js
module.exports = app => {
  app.router.post('/form', app.controller.form.post);
}

// app/controller/form.js
module.post = async ctx => {
  ctx.body = `body: ${JSON.stringify(ctx.request.body)}`;
}

// curl -X POST http://127.0.0.1:7001/form --data '{"name":"controller"}' --header 'Content-Type:application/json'
```

> 附：

> 這裡直接發起 POST 請求會報錯：'secret is missing'。錯誤信息來自 koa-csrf/index.js#L69

> 原因：框架內部針對 POST 請求均會驗證 CSRF 的值，因此我們在表單提交時，請帶上 CSRF key 進行提交，可參考[安全威嚇csrf的防範](https://eggjs.org/zh-cn/core/security.html#%E5%AE%89%E5%85%A8%E5%A8%81%E8%83%81csrf%E7%9A%84%E9%98%B2%E8%8C%83)

> 注意：上面的效驗是因為框架中內置了安全插件 egg-security，提供了一些默認的安全實踐，並且框架的安全插件是默認開啟的，如果需要關閉其中一些安全防範，直接設置該項的 enable 屬性為 false 即可。

> 「除非清楚確認結果，反則不建議關閉安全插件提供的功能。」

> 可臨時在 `config/config.default.js` 設置：

```javascript
exports.security = {
  csrf: false
};
```

### 表單效驗

```javascript
// app/router.js
module.exports = app => {
  app.router.post('/user', app.controller.user);
};

// app/controller/user.js
const createRule = {
  username: {
    type: 'email',
  },
  password: {
    type: 'password',
    compare: 're-password',
  },
};

exports.create = async ctx => {
  // 如果效驗報錯，會拋出異常
  ctx.validate(createRule);
  ctx.body = ctx.request.body;
};

// curl -X POST http://127.0.0.1:7001/user --data 'username=abc@abc.com&password=111111&re-password=111111'
```

### 重定向

#### 內部重定向

```javascript
// app/router.js
module.exports = app => {
  app.router.get('index', '/home/index', app.controller.home.index);
  app.router.redirect('/', '/home/index', 302);
};

// app/controller/home.js
exports.index = async ctx => {
  ctx.body = 'hello controller';
};

// curl -L http://localhost:7001
```

#### 外部重定向

```javascript
// app/router.js
module.exports = app => {
  app.router.get('/search', app.controller.search.index);
};

// app/controller/search.js
exports.index = async ctx => {
  const type = ctx.query.type;
  const q = ctx.query.q || 'nodejs';

  if (type === 'bing') {
    ctx.redirect(`http://cn.bing.com/search?q=${q}`);
  } else {
    ctx.redirect(`https://www.google.co.kr/search?q=${q}`);
  }
};

// curl http://localhost:7001/search?type=bing&q=node.js
// curl http://localhost:7001/search?q=node.js
```

### 中間件的使用

如果我們想把用戶某一類請求的參數都大寫，可透過中間件來實現。

```javascript
// app/controller/search.js
exports.index = async ctx => {
  ctx.body = `search: ${ctx.query.name}`;
};

// app/middleware/uppercase.js
module.exports = () => {
  return async function uppercase(ctx, next) {
    ctx.query.name = ctx.query.name && ctx.query.name.toUpperCase();
    await next();
  };
};

// app/router.js
module.exports = app => {
  app.router.get('s', '/search', app.middlewares.uppercase(), app.controller.search)
};

// curl http://localhost:7001/search?name=egg
```

### 太多路由映射？

我們並不建議將路由規則邏輯散落在多地方，會有排查問題。

若確實有需求，可以如下拆分：


```javascript
// app/router.js
module.exports = app => {
  require('./router/news')(app);
  require('./router/admin')(app);
};

// app/router/news.js
module.exports = app => {
  app.router.get('/news/list', app.controller.news.list);
  app.router.get('/news/detail', app.controller.news.detail);
};

// app/router/admin.js
module.exports = app => {
  app.router.get('/admin/user', app.controller.admin.user);
  app.router.get('/admin/log', app.controller.admin.log);
};
```