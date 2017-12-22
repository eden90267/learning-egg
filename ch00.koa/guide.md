# 指南

本指南涵蓋主題不與 API 直接相關，例如編寫中間件的最佳做法和應用程序結構建議。在這些例子中，我們使用 async 函數作為中間件 - 你也可以使用 commonFunction 或 generatorFunction，這將些所不同。

## 編寫中間件

Koa 中間件是簡單的函數，它返回一個帶有簽名 (ctx, next) 的 MiddlewareFunction。當中間件運行時，它必須手動調用 next() 來運行 “下游” 中間件。

例如，如果你想要跟蹤透過添加 X-Response-Time 頭字段透過 Koa 傳播請求需要多長時間，則中間件將如下所示：

```javascript
async function responseTime(ctx, next) {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
}

app.use(responseTime);
```

如果您是前端開發人員，您可以將 `next();` 之前的任意代碼視為“捕獲”階段，下面簡易的 git 說明了 async 函數如何使我們能夠恰當地利用堆棧來實現請求和響應流：


![](https://raw.githubusercontent.com/demopark/koa-docs-Zh-CN/master/middleware.gif)

接下來，將介紹創建 Koa 中間件的最佳做法。

## 中間件最佳實踐

本節介紹中間件創作最佳實踐，例如中間件接受參數，命名中間件進行調試等等。

### 中間件參數

當創建公用中間件時，將中間件包裝在接受參數的函數中，遵循這個約定是有用的，允許用戶擴展功能。即便您的中間件 _不_ 接受任何參數，這仍然是保持統一的好方法。

這裡我們設計的 logger 中間件接受一個 format 自定義字符串，並返回中間件本身：

```javascript
function logger(format) {
  format = format || ':method ":url"';
  
  return async (ctx, next) => {
    const str = format
      .replace(':method', ctx.method)
      .replace(':url', ctx.url);
    
    console.log(str);
    
    await next();
  }
}

app.use(logger());
app.use(logger(':method :url'));
```

### 命名中間件

命名中間件是可選的，但在調試中分配名稱很有用。

```javascript
function logger(format) {
  return async function logger(ctx, next) {
    
  };
}
```

### 將多個中間件與 koa-compose 相結合

有時你想要將多個中間件 “組合” 成一個單一的中間件，便於重用或導出。你可使用 [koa-compose](https://github.com/koajs/compose)

```javascript
const compose = require('koa-compose');

async function radom(ctx, next) {
  if ('/random' == ctx.path) {
    ctx.body = Math.floor(Math.random * 10);
  } else {
    await next();
  }
}

async function backwards(ctx, next) {
  if ('/backwards' == ctx.path) {
    ctx.body = 'sdrawkcab';
  } else {
    await next();
  }
}

async function pi(ctx, next) {
  if ('/pi' == ctx.path) {
    ctx.body = String(Math.PI);
  } else {
    await next();
  }
}

const all = compose([random, backwards, pi]);
app.use(all);
```

### 響應中間件

中間件決定響應請求，並希望繞過下游中間件可以簡單地省略 `next()`。通常這將在路由中間件中，但這也可以任意執行。例如，以下內容將以 ”two“ 進行響應，但是所有三個都將被執行，從而使下游的 “three” 中間件有機會操縱響應。

```javascript
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  await next();
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

以下配置在第二個中間件中省略了 next()，並且仍然會以 “two” 進行響應，然而，第三個 (以及任何其他下游中間件) 將被忽略：

```javascript
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

當最遠的下游中間件執行 `next();` 時，它實際上是一個 noop 函數，允許中間件在堆棧中的任意位置正確組合。

## 異步操作

Async 方法和 promise 來自 Koa 的底層，可以讓你編寫非阻塞序列代碼。例如，這個中間件從 `./docs` 讀取文件名，然後再將給 body 指定合併結果之前並行讀取每個 markdown 文件的內容。

```javascript
const fs = require('fs-promise');

app.use(async function (ctx, next) {
  const paths = await fs.readdir('docs');
  const files = await Promise.all(paths.map(path => fs.readFile(`doc/${path}`, 'utf-8')));
  
  ctx.type = 'markdown';
  ctx.body = files.join('');
});
```

## 調試 Koa

Koa 以及許多構建庫，支持來自 [debug](https://github.com/visionmedia/debug) 的 DEBUG 環境變量，它提供簡單的條件紀錄。

例如，要查看所有 koa 特定的調試信息，只需透過 DEBUG=koa*，並且在啟動時，您將看到所使用的中間件的列表。

```shell
$ DEBUG=koa* node --harmony examples/simple
  koa:application use responseTime +0ms
  koa:application use logger +4ms
  koa:application use contentLength +0ms
  koa:application use notfound +0ms
  koa:application use response +0ms
  koa:application listen +0ms
```

由於 JavaScript 在運行時沒有定義函數名，你也可以將中間件的名稱設置為 `._name`。當你無法控制中間件的名稱時，這很有用。例如：

```javascript
const path = require('path');
const serve = require('koa-static');

const publicFiles = serve(path.join(__dirname, 'public'));
publicFiles._name = 'static /public';

app.use(publicFiles);
```

現在，在調試時不只或看到 “serve”，你也會看到：

```shell
koa:application use static /public +0ms
```