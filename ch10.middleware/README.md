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

- options：
- app：
