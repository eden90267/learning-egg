# Middleware

Egg 的中間件形式和 Koa 中間件形式是一樣的，都是基於洋蔥圈模型。每次我們編寫一個中間件，就相當在洋蔥外妙包一層。

## 編寫中間件

### 寫法

我們先編寫一個簡單的 gzip 中間件，來看看中間件寫法。

```javascript
// app/middleware/gzip.js
const isJSON = require('koa-is-json');
const zlib = require('zlib');

async function gzip(ctx, next) {
  await next();
  
  let body = ctx.body;
  if (!body) return;
  if (isJSON(body)) body = JSON.stringify(body);
  
  const stream = zlib.createGzip();
  stream.end(body);
  ctx.body = stream;
  ctx.set('Content-Encoding', 'gzip');
}
```

可看到，框架的中間件和 Koa 的中間件寫法是一模一樣的，所以任何 Koa 的中間件都可以直接被框架使用。

### 配置

一般來說中間件也會有自己的設置。在框架中，一個完整的中間件是包含了配置處理的。我們約定一個中間件是一個放置在 `app/middleware` 目錄下的單獨文件，它需要 exports 一個普通 function，接受兩個參數：

- options：中間件的配置項，框架會將 `app.config[${middlewareName}]` 傳遞進來
- app：當前應用 Application 的實例

我們將上面的 gzip 中間件做一個簡單的優化，讓它支持指定只有當 body 大於配置的 threshold 時才進行 gzip 壓縮：

```javascript
const isJSON = require('koa-is-json');
const zlib = require('zlib');

module.exports = options => {
  return async function gzip(ctx, next) {
    await next();
    
    let body = ctx.body;
    if (!body) return;
    
    if (options.threshold && ctx.length > options.threshold) return;
    
    if (isJSON(body)) body = JSON.stringify(body);
    
    const stream = zlib.createGzip();
    stream.end(body);
    ctx.body = stream;
    ctx.set('Content-Encoding', 'gzip');
  }
}
```

## 使用中間件

中間件編寫完成後，我們還需要手動掛載，支持以下方式：

### 在應用中使用中間件

在應用中，我們可以完全透過配置來加載自定義的中間件，並決定它們的順序。

在 `config.default.js` 中加入下面配置就完成了中間件的開啟和配置：

```javascript
module.exports = {
  // 配置需要的中間件，數組順序即為中間件的加載順序
  middleware: ['gzip'],
  
  // 配置 gzip 中間件的配置
  gzip: {
    threshold: 1024, // 小於 1k 的響應體不壓縮
  },
};
```

該配置最終將在啟動時合併到 `app.config.appMiddleware`。

### 在框架和插件中使用中間件

框架和插件不支持在 `config.default.js` 中匹配 `middleware`，需要透過以下方式：

```javascript
// app.js
module.exports = app => {
  // 在中間件最前面統計請求時間
  app.config.coreMiddleware.unshift('report');
}

// app/middleware/report.js
module.exports = () => {
  return async function report(ctx, next) {
    const startTime = Date.now();
    await next();
    // 上報請求時間
    reportTime(Date.now() - startTime);
  }
}
```

應用層定義的中間件 (`app.config.addMiddleware`) 和框架默認中間件 (`app.config.coreMiddleware`) 都會被加載器加載，並掛載到 `app.middleware` 上。

### router 中使用中間件

以上兩種方式配置的中間件是全局的，會處理每一次請求，如果你只想針對單個路由生效，可以直接在 `app/router.js` 中實例化和掛載，如下：

```javascript
module.exports = app => {
  const gzip = app.middlewares.gzip({threshold: 1024});
  app.router.get('/needgzip', gzip, app.controller.handler);
}
```

## 框架默認中間件

除了應用層加載中間件之外，框架和自身其他插件也會加載許多中間件。所有這些自帶中間件的配置項都透過在配置中修改中間件同名配置項進行修改，例如[框架自帶的中間件](https://github.com/eggjs/egg/tree/master/app/middleware)中有一個 bodyParser 中間件 (框架的加載器會將文件名中的各種分隔符都修改成駝峰形式的變量名)，我們想要修改 bodyParser 配置，只需在 `config/config.default.js` 中編寫：

```javascript
module.exports = {
  bodyParser: {
    jsonLimit: '10mb',
  }
}
```

注意：框架和插件加載的中間件會在應用層配置的中間件之前，框架默認中間件不能被應用層中間件覆蓋，如果應用層有自定義同名中間件，在啟動時會報錯。

## 使用 Koa 的中間件

在框架裡面可以非常容易的引入 Koa 中間件生態。

以 koa-compress 為例，在 Koa 中使用時：

```javascript
const koa = require('koa');
const compress = require('koa-comress');

const app = koa();

const options = {threshold: 2048};
app.use(compress(options));
```

我們按照框架的規範來在應用中加載這個 Koa 的中間件：

```javascript
// app/middleware/compress.js
// koa-compress 暴露的接口 (`(options) => middleware`) 和框架對中間件要求一致
module.exports = require('koa-compress');
```

```javascript
// config/config.default.js
module.exports = {
  middleware: ['compress'],
  compress: {
    threshold: 2048,
  }
}
```

如果使用到的 Koa 中間件不符合入參規範，則可以自行處理下：

```javascript
// config/config.default.js
module.exports = {
  webpack: {
    compiler: {},
    others: {}
  }
}

// app/middleware/webpack.js
const webpackMiddleware = require('some-koa-middleware');

module.exports = (options, app) => {
  return webpackMiddleware(options.compiler, options.others);
}
```

## 通用配置

無論是應用層加載的中間件還是框架自帶中間件，都支持幾個通用的配置項：

- enable：控制中間件是否開啟
- match：設置只有符合某些規則的請求才會經過這個中間件
- ignore：設置符合某些規則的請求不經過這個中間件

### enable

```javascript
module.exports = {
  bodyParser: {
    enable: false,
  },
};
```

### match 和 ignore

match 和 ignore 支持的參數都一樣，只是作用相反，match 和 ignore 不允許同時配置。

```javascript
module.exports = {
  gzip: {
    match: '/static',
  },
};
```

match 和 ignore 支持多種類性的配置方式：

1. 字符串：當參數是字符串類型，配置的是一個 url 的路徑前綴，所有以配置的字符串作為前綴的 url 都會匹配上
2. 正則：當參數是正則時，直接匹配滿足正則驗證的 url 的路徑
3. 函數：當參數是一個函數時，會將請求上下文傳遞給這個函數，最終取函數返回的結果 (true / false) 來判斷是否匹配

```javascript
module.exports = {
  gzip: {
    match(ctx) {
      // 只有 ios 設備才開啟
      const reg = /iphone|ipad|ipod/i;
      return reg.test(ctx.get('user-agent'));
    },
  },
};
```