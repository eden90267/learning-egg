# Egg.js 與 Koa

## 異步編程模型

Node.js 是一個異步的世界，官方 API 支持的都是 callback 形式的異步編程模型，這會帶來許多問題，例如：

- callback hell：最臭名昭著的 callback 嵌套問題
- release zalgo：異步函數中可能同步調用 callback 返回資料，帶來不一致性

因此社區提供各種異步的解決方案，最終勝出的是 Promise，它也內置到 ECMAScript 2015 中。而在 Promise 的基礎上，結合 Generator 提供的切換上下文能力，出現了 co 等第三方類庫來讓我們用同步寫法編寫異步代碼。同時，async function 這個官方解決方案也於 ECMAScript 2017 中發布，並在 Node.js 8 中實現。

## async function

async function 是語言層面提供的語法糖，在 async function 中，我們可以通過 await 關鍵字來等待一個 Promise 被 resolve (或者 reject，此時會拋出異常)，Node.js 現在的 LTS 版本 (8.x) 已原生支持。

```javascript
const fn = async function() {
  const user = await getUser();
  const posts = await fetchaPosts(user.id);
  return {user, posts};
}
fn().then(res => console.log(res)).catch(err => console.error(err.stack));
```

## Koa

> Koa is a new Web framework designed by the team behind Express, which aims to be a smaller, more expressive, and more robust foundation for Web applications and APIs.

Koa 和 Express 的設計風格非常類似，底層也都是共用的同一套 HTTP 基礎庫，但是有幾個顯著的區別，除了上面提到的默認異步解決方案之外，主要的特點還有下面幾個。

### Middleware

Koa 的中間件和 Express 不同，Koa 選擇了洋蔥圈模型。

- 中間件洋蔥圖：

![](https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67)

- 中間件執行順序圖：

![](https://raw.githubusercontent.com/koajs/koa/a7b6ed0529a58112bac4171e4729b8760a34ab8b/docs/middleware.gif)

所有請求經過一個中間件的時候都會執行兩次，對比 Express 形式的中間件，Koa 的模型可以非常方便的實現後製處理邏輯，對比 Koa 和 Express 的 Compress 中間件就可以明憲的感受到 Koa 中間件模型的優勢

- [koa-compress](https://github.com/koajs/compress/blob/master/index.js) for Koa
- [compression](https://github.com/expressjs/compression/blob/master/index.js) for Express

### Context

和 Express 只有 Request 和 Response 兩個對象不同，Koa 增加了一個 Context 的對象，作為這是請求的上下文對象 (在 Koa 1 中為中間件的 this，在 Koa 2 中作為中間件的第一個參數傳入)。我們可以將一次請求相關的上下文都掛載到這個對象上。類似 traceId 這種需要貫穿整個請求 (在後續任何一個地方進行其他調用都需要用到)的屬性就可以掛載上去。相較於 request 和 response 而言更加符合語意。

同時 Context 上也掛載了 Request 和 Response 兩個對象。和 Express 類似，這兩個對象都提供了大量的便捷方法補助開發，例如：

- get request.query
- get request.hostname
- set response.body
- set response.status

### 異常處理

透過同步方式編寫異步代碼帶來的另外一個非常大的好處就是異常處理非常自然，使用 try catch 就可以將按照規範編寫的代碼中的所有錯誤都補獲到。這樣我們可以很便捷的編寫一個自定義的錯誤處理中間件。

```javascript
async function onerror(ctx, next) {
  try {
    await next();
  } catch (err) {
    ctx.app.emit('error', err);
    ctx.body = 'server error';
    ctx.status = err.status || 500;
  }
}
```

只需要將這個中間件放在其他中間件之前，就可以補獲它們所有的同步或者異步代碼中拋出的異常了。

## Egg 繼承自 Koa

Koa 是一個非常優秀的框架，但對企業級應用來說，它還比較基礎。

而 Egg 選擇了 Koa 作為基礎框架，在它的模型基礎上，近一步對他進行了一些增強。

### 擴展

基於 Egg 的框架或應用，我們可透過定義 `app/extend/{application,context,request,response}.js` 來擴展 Koa 中對應的四個對象的原型，透過這個功能，我們可以快速地增加更多的輔助方法，例如 app/extend/context.js 中寫入下列代碼：

```javascript
module.exports = {
  get isIOS() {
    const iosReg = /iphone|ipad|ipod/i;
    return iosReg.test(this.get('user-agent'));
  },
};
```

在 Controller 中，我們就可以使用到剛才定義的這個便捷屬性了：

```javascript
exports.handler = ctx => {
  ctx.body = ctx.isIOS 
    ? 'your operating system is iOS'
    : 'Your operating system is not iOS';
}
```

### 插件

在 Express 和 Koa 中，經常會引入許許多多的中間件來提供各種各樣的功能，例如引入 koa-session 提供 Session 的支持，引入 koa-bodyparser 來解析請求 body。而 Egg 提供了一個更加強大的插件機制，讓這些獨立領域的功能模組可以更加容易編寫。

一個插件可以包含：

- extend：擴展基礎對象的上下文，提供各種工具類、屬性
- middleware：增加一個或多個中間件，提供請求的前置、後置處理邏輯
- config：配置各個環境下插件自身的默認配置項

一個獨立領域下的插件實現，可以在代碼維護性非常高的情況下實現非常完善的功能，而插件也支持配置各個環境下的默認 (最佳) 配置，讓我們使用插件的時候可以不需要修改配置項。

egg-security 插件就是一個典型的例子。

### Egg 與 Koa 的版本關係

Egg 1.x 基於 Koa 1.x 開發，但全面增加 async function 的支持，且 Egg 對 Koa 2.x 的中間件也完全兼容，應用層代碼可以完全基於 async function 來開發。

- 底層基於 Koa 1.x，異步解決方案基於 co 封裝的 generator function
- 官方插件以及 Egg 核心使用 generator function 編寫，保持對 Node.js LTS 版本的支持，在必要處通過 co 包裝以兼容在 async function 中的使用
- 應用開發者可選擇 async function (Node.js 8.x+) 或者 generator function (Node.js 6.x+) 進行編寫

### Egg 2.x

Node.js 8 正式進入 LTS 後，async function 可以在 Node.js 中使用並且沒有任何性能問題了，Egg 2.x 基於 Koa 2.x，框架底層以及所有內置插件都可使用 async function 編寫，並保持了對 Egg.js 以及 generator function 的完全兼容，應用層只需要升級到 Node.js 8 即可從 Egg 1.x 遷移到 Egg 2.x。

- 底層基於 Koa 2.x，異步解決方案基於 async function
- 官方插件以及 Egg 核心使用 async function
- 建議業務層遷移到 async function 方案
- 只支持 Node.js 8 及以上的版本


