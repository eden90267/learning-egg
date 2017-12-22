# 錯誤處理

## Try-Catch

使用 async 方法意味著你可以 try-catch `next`。此示例為所有錯誤添加一個 `.status`：

```javascript
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (err) {
    err.status = err.statusCode || err.status || 500;
    throw err;
  }
});
```

### 默認錯誤處理程序

默認的錯誤處理程序本質上是中間件鏈開始時的一個 try-catch。要使用不同的錯誤處理程序，只需在中間件鏈的起始處放置另一個 try-catch，並在那裡處理錯誤。但是，默認錯誤處理程序對於大多數用例來說都是足夠好的。它將使用狀態代碼 `err.status`，或默認為 500。如果 `err.expose` 是 true，那麼 `err.message` 就是答覆。否則，將使用從錯誤代碼生成的消息 (例如代碼 500，就是 “內部服務器錯誤”)。所有標頭將從請求中清除，但是任何在 `err.headers` 中的標頭將會被設置。你可使用如上所述的 try-catch 來向此列表添加標頭：

以下是創建你自己的錯誤處理程序的示例：

```javascript
app.use(async (ctx, next) => {
  try {
    await next();
  } catch(err) {
    // will only response with JSON
    ctx.status = err.statusCode || err.status || 500;
    ctx.body = {
      message: err.message
    };
  }
})
```

## 錯誤事件

錯誤事件監聽器可以用 `app.on('error')` 指定。如果未指定錯誤監聽器，則使用默認錯誤監聽器。錯誤監聽器接收所有中間件鏈返回的錯誤，如果一個錯誤被捕獲並且不再拋出，他將不會被傳遞給錯誤監聽器。如果沒有指定錯誤事件監聽器，那麼將使用 `app.onerror`，如果 `error.expose` 為 true 並且 `app.silent` 為 false，則簡單記錄錯誤。