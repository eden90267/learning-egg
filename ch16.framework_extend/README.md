# 框架擴展

框架提供了多種擴展點擴展自身的功能

- Application
- Context
- Request
- Response
- Helper

在開發中，我們既可使用已有的擴展 API 來方便開發，也可對以上對象進行自定義擴展，近一步加強框架功能。

## Application

app 對象指的是 Koa 的全局應用對象，全局只有一個，在應用啟動時被創建。

###  訪問方式

- `ctx.app`
- Controller, Middleware, Helper, Service 中都可透過 `this.app` 訪問到 Application 對象，例如 `this.app.config` 訪問配置對象。
- 在 `app.js` 中 app 對象會作為第一個參數注入到入口函數中

```javascript
// app.js
module.exports = app => {
  // 使用 app 對象
};
```

### 擴展方式

框架會把 `app/extend/application.js` 中定義的對象與 Koa Application 的 prototype 對象進行合併，在應用啟動時會基於擴展後的 prototype 生成 app 對象。

#### 方法擴展

```javascript
// app/extend/application.js
module.exports = {
  foo(param) {
    // this 就是 app 對象，可調用其他方法或屬性
  }
};
```

#### 屬性擴展

一般來說屬性的計算只需進行一次，那麼一定要實現緩存，否則在多次訪問屬性時會計算多次，這樣會降低應用性能。

推薦使用 Symbol + Getter 的模式。

例如，增加一個 `app.bar` 屬性 Getter：

```javascript
// app/extend/application.js
const BAR = Symbol('Application#bar');

module.exports = {
  get bar() {
    if (!this[BAR]) {
      this[BAR] = this.config.xx + this.config.yy;
    }
    return this[BAR];
  }
}
```

## Context

Context 是 Koa 的請求上下文，這是 **請求級別** 的對象，每次請求生成一個 Context 實例。

### 訪問方式

- middleware 中 this 就是 ctx
- controller 有兩種寫法，類：`this.ctx`，方法：ctx 入參
- helper, service 中的 this 指向 helper, service 對象本身，使用 `this.ctx` 訪問 context 對象

### 擴展方式

框架會把 `app/extend/context.js` 中定義的對象與 Koa Context 的 prototype 對象進行合併，在處理請求時會基於擴展後的 prototype 生成 ctx 對象。

#### 方法擴展

```javascript
// app/extend/context.js
module.exports = {
  foo(param) {
    // this 就是 ctx 對象
  }
}
```

#### 屬性擴展

一般來說屬性的計算只需進行一次，那麼一定要實現緩存，否則在多次訪問屬性時會計算多次，這樣會降低應用性能。

推薦使用 Symbol + Getter 的模式。

例如，增加一個 `ctx.bar` 屬性 Getter：

```javascript
// app/extend/context.js
const BAR = Symbol('Context#bar');

module.exports = {
  get bar() {
    if (!this[BAR]) {
      // 例如，從 header 中獲取，實際情況肯定更複雜
      this[BAR] = this.get('x-bar');
    }
    return this[BAR];
  },
};
```

## Request

Request 對象和 Koa 的 Request 對象相同，是 **請求級別** 的對象，它提供了大量請求相關的屬性和方法供使用

### 訪問方式

```javascript
ctx.request
```

ctx 上很多屬性和方法都被代理到 request 對象上。

Koa 內置的代理 request 的屬性和方法列表：[Koa - Request aliases](http://koajs.com/#request-aliases)

### 擴展方式

框架會把 `app/extend/request.js` 中定義的對象與內置 request 的 prototype 對象進行合併，在處理請求時會基於擴展後的 prototype 生成 request 對象。

例如，增加一個 `request.foo` 屬性 Getter：

```javascript
// app/extend/request.js
module.exports = {
  get foo() {
    return this.get('x-request-foo');
  }
}
```

## Response

Response 對象和 Koa 的 Response 對象相同，是 **請求級別** 的對象，它提供了大量請求相關的屬性和方法供使用

### 訪問方式

```javascript
ctx.response
```

ctx 上很多屬性和方法都被代理到 response 對象上。

Koa 內置的代理 response 的屬性和方法列表：[Koa - Response aliases](http://koajs.com/#response-aliases)

### 擴展方式

框架會把 `app/extend/response.js` 中定義的對象與內置 response 的 prototype 對象進行合併，在處理請求時會基於擴展後的 prototype 生成 response 對象。

例如，增加一個 `response.foo` 屬性 Setter：

```javascript
// app/extend/response.js
module.exports = {
  set foo(value) {
    this.set('x-response-foo', value);
  },
};
```

就可以這樣使用：`this.response.foo = 'bar';`

## Helper

Helper 函數用來提供一些實用的 utility 函數

它的作用在於我們可將一些常用的動作抽離在 helper.js 裡面成為一個獨立的函數。

框架內置一些常用的 Helper 函數，我們也可編寫自定義的 Helper 函數

### 訪問方式

```javascript
// 假設在 app/router.js 中定義了 home router
app.get('home', '/', 'home.index');

// 使用 helper 計算指定 url path
ctx.helper.pathFor('home', {by: 'recent', limit: 20});
// => /?by=recent&limit=20
```

### 擴展方式

框架會把 `app/extend/helper.js` 中定義的對象與內置 helper 的 prototype 對象進行合併，在處理請求時會基於擴展後的 prototype 生成 helper 對象。

例如，增加一個 `helper.foo()` 方法：

```javascript
// app/extend/helper.js
module.exports = {
  foo(param) {
    // this 是 helper 對象，在其中可以調用其他 helper 方法
    // this.ctx => context 對象
    // this.app => application 對象
  }
}
```

## 按照環境進行擴展

另外，還可根據環境進行有選擇的擴展，例如，只在 unittest 環境中提供 `mockXX()` 方法以便進行 mock 方便測試。

```javascript
// app/extend/application.unittest.js
module.exports = {
  mockXX(k, v) {
  }
};
```

這隻文件只會在 unittest 環境加載

對於 Application、Context、Request、Response、Helper 都可使用這種方式針對某個環境進行擴展，更多參見[運行環境](https://eggjs.org/zh-cn/basics/env.html)。