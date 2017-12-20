# 升級指南

## 背景

隨著 Node.js 8 LTS 的發布，內建了對 ES 2017 Async Function 的支持

在這之前，TJ 的 co 使我們可以提前享受到 `async/await` 的編程體驗，但同時它不可避免的也帶來一些問題：

- 性能損失
- 錯誤堆棧不友好

現在 Egg 正式發布 2.x 版本：

- 保持對 Egg 1.x 以及 generator function 的完全兼容
- 基於 Koa 2.x，異步解決方案基於 async function
- 只支持 Node.js 8 及以上版本
- 去除 co 後堆棧信息更清晰，帶來 30% 左右的性能提升 (不含 Node 帶來的性能提升)

Egg 的理念之一是 漸進式增強，故我們為開發者提供 漸進升級 的體驗：

- [快速升級](https://eggjs.org/zh-cn/migration.html#%E5%BF%AB%E9%80%9F%E5%8D%87%E7%BA%A7)
- [插件變更說明](https://eggjs.org/zh-cn/migration.html#%E6%8F%92%E4%BB%B6%E5%8F%98%E6%9B%B4%E8%AF%B4%E6%98%8E)
- [進一步升級](https://eggjs.org/zh-cn/migration.html#%E8%BF%9B%E4%B8%80%E6%AD%A5%E5%8D%87%E7%BA%A7)

  - 修改為推薦的代碼風格
  - 中間件使用 Koa2 風格
  - 函數調用的 yieldable 轉為 awaitable

- [針對插件開發者的升級指南](https://eggjs.org/zh-cn/migration.html#%E6%8F%92%E4%BB%B6%E5%8D%87%E7%BA%A7)

(就不詳列了，因為筆者根本沒用過 egg 1.x ...)

---

參考：[https://eggjs.org/zh-cn/migration.html#%E5%BF%AB%E9%80%9F%E5%8D%87%E7%BA%A7](https://eggjs.org/zh-cn/migration.html#%E5%BF%AB%E9%80%9F%E5%8D%87%E7%BA%A7)