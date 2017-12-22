# 常見問題

## Koa 替代 Express?

它更像是 Connect，但是很多 Express 的好東西被移轉到 Koa 的中間件級別，以幫助形成更強大的基礎。這使得中間件對於整個堆棧而言不僅僅是最終應用程序代碼，而且更易於書寫，並更不容易出錯。

通常，許多中間件將重新實現類似的功能，甚至更糟的是不正確地實現它們，如簽名的 cookie 加密等通常是應用程序特定的，而不是中間件特定的。

## Koa 替代 Connect?

不，只是不同的功能，現在透過構建器也可以讓我們用較少的回調編寫代碼。Connect 同樣可以，有些人可能仍然喜歡它，這取決於你喜歡什麼。

## Koa 包含路由嗎?

不 - Koa 沒有開箱即用的路由，但是有很多路由中間件可用：[https://github.com/koajs/koa/wiki](https://github.com/koajs/koa/wiki)

## 為什麼 Koa 不是 Express 4.0?

Koa 與現在所知的 Express 差距很大，設計根本上有很大差異，所以從 Express 3.0 遷移到 Express 4.0 將有意謂著重寫整個應用程序，所以我們考慮創建一個新的庫。

## Koa 對象有什麼自定義屬性?

Koa 使用它的自定義對象：ctx、ctx.request、和 ctx.response。這些對象使用便捷的方法和 getter/setter 來抽象 node 的 req 和 res 對象。

通常，添加到這些對象的屬性必須遵循以下規則：

- 它們必須是非常常用的 和/或 必須做一些有用的事情
- 如果一個屬性作為一個 setter 存在，那麼它也將做為一個 getter 存在，但反之亦然

許多 ctx.request 和 ctx.response 的屬性都被委託給 ctx。如果它是一個 getter/setter，那麼 getter 和 setter 都將嚴格對應於 ctx.request 或 ctx.response。

附加其他屬性之前，請考慮這些規則。