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

## Koa 替代 Express?

它更像是 Connect，但是很多 Express 的好東西被移轉到 Koa 的中間件級別，以幫助形成更強大的基礎。這使得中間件對於整個堆棧而言不僅僅是最終應用程序代碼，而且更易於書寫，並更不容易出錯。

通常，許多中間件將重新實現類似的功能，甚至更糟的是不正確地實現它們，如簽名的 cookie 加密等通常是應用程序特定的，而不是中間件特定的。

## Koa 替代 Connect?

不，只是不同的功能，現在透過構建器也可以讓我們用較少的回調編寫代碼。Connect 同樣可以，有些人可能仍然喜歡它，這取決於你喜歡什麼。

## 為什麼 Koa 不是 Express 4.0?

Koa 與現在所知的 Express 差距很大，設計根本上有很大差異，所以從 Express 3.0 遷移到 Express 4.0 將有意謂著重寫整個應用程序，所以我們考慮創建一個新的庫。

## Koa 與 Connect/Express 有哪些不同?

### 基於 Promises 的控制流程

沒有回調地獄。

透過 try/catch 更好的處理錯誤。

無需域。

### Koa 非常精簡

不同於 Connect 和 Express, Koa 不含任何中間件。

不同於 Express，不提供路由。

不同於 Express，不提供許多便捷設施。例如，發送文件。

Koa 更加模組化。

### Koa 對中間件的依賴較少

例如，不使用 “body parsing” 中間件，而是使用 body 解析函數。

### Koa 抽象 node 的 request/response

減少攻擊。

更好的用戶體驗。

恰當的流處理。

### Koa 路由 (第三方庫支持)

由於 Express 帶有自己的路由，而 Koa 沒有任何內置路由，但是有 koa-router 和 koa-route 第三方庫可用。同樣的，就像我們在 Express 中有 helmet 保證安全，對於 koa 我們有 koa-helmet 和一些列的第三方庫可用。