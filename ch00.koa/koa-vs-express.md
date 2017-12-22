# Koa 與 Express

在理念上，Koa 旨在 “修復和替換節點”，而 Express 旨在 “增加節點”。Koa 使用 Promise 和 async 功能來擺脫 callback hell 的應用程序，並簡化錯誤處理。它暴露了自己的 ctx.request 和 ctx.response 對象，而不是 node 的 req 和 res 對象。

另一方面，Express 透過附加的屬性和方法增加了 node 的 req 和 res 對象，並且包含許多其他 “框架” 功能，如 路由 和 模板，而 Koa 則沒有。

因此，Koa 可被視為 node.js 的 http 模組的**抽象**，其中 Express 是 node.js 的應用程序框架。

| 功能               | Koa | Express | Connect |
|:------------------|:----|:--------|:--------|
| Middleware Kernel | ✓   | ✓       | ✓       |
| Routing           |     | ✓       |         |
| Templating        |     | ✓       |         |
| Sending File      |     | ✓       |         |
| JSONP             |     | ✓       |         |

因此，如果你想要**更接近 node.js 和傳統的 node.js 樣式編碼**，那麼你可能希望堅持使用 Connect/Express 或類似的框架。如果你想要**擺脫回調**，請使用 Koa。

由於這種不同的理念，其結果是傳統的 node.js “中間件” (即 “(req, res, next)” 的函數) 與 Koa 不兼容。你的應用基本上要重新改寫了。