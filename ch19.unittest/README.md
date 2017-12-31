# 單元測試

## 為什麼要單元測試

先問自己以下幾個問題：

- 你的代碼質量如何度量？
- 你是如何保證代碼質量？
- 你敢隨時重構代碼嗎？
- 你是如何確保重構的代碼依然保持正確性？
- 你是否有足夠信心在沒有測試情況下隨時發佈你的代碼？

如果答案都比較猶豫，那麼就證明我們非常需要單元測試

它能帶給我們很多保障：

- 代碼質量持續有保障
- 重構正確性保障
- 增強自信心
- 自動化運行

Web 應用中的單元測試更加重要，在 Web 產品快速迭代的時期，每個測試用例都給應用的穩定性提供了一層保障。

- API 升級，測試用例可以很好地檢查代碼是否向下兼容。
- 對於各種可能的輸入，一旦測試覆蓋，都能明確它的輸出。
- 代碼改動後，可以透過測試結果判斷代碼的改動是否影響已確定的結果。

所以，應用的 Controller、Service、Helper、Extend 等代碼，都必須有對應的單元測試保證代碼質量。當然，框架和插件的每個功能改動和重構都需要有相應的單元測試，並且要求盡量做到修改的代碼能被 100% 覆蓋到。

## 測試框架

### Mocha

我們推薦跟選擇 Mocha，功能非常豐富，支持運行在 Node.js 和瀏覽器中，對異步測試支持非常友好。

> Mocha is a feature-rich JavaScript test framework running on Node.js and in the browser, making asynchronous testing simple and fun. Mocha tests run serially, allowing for flexible and accurate reporting, while mapping uncaught exceptions to the correct test cases.

### AVA

我們並未選擇最近比較火的 AVA，它看起來會跑的很快。但經過幾個真實項目實踐後，AVA 真的只是看起來很美，但是實際上會讓測試代碼越來越難寫，成本越來越高。

@dead-horse 的評價：

- AVA 自身不夠穩定，運行文件多會撐爆 CPU；如果設置控制併發參數的方式運行，會導致 only 模式無效。
- 併發執行對測試用例的要求很高，所有測試不能有依賴，特別是遇到一些需要做 mock 的場景時，寫好很難。
- app 在初始化的時候是有耗時的，如果串行運行，只需要初始化一個 app 對它測試。但是 AVA 每一個文件就需要初始化多少個 app。

@fool2fish 的評價：

如果是簡單程序用 AVA 會快一些，如果是複雜的就不推薦了，比較大的問題是可能沒法給出準確的錯誤堆棧，另外併發可能會導致依賴的其他測試環境的服務掛掉，降低測試的成功率，還有就是帶流程測試真心不適合用 AVA。

## 斷言庫

同樣，測試斷言庫也是百花齊放的時代，我們經歷 assert，到 should 和 expect，還是不斷地嘗試更好的斷言庫。

直到我們發現 [power-assert](https://github.com/power-assert-js/power-assert)，因為『No API is the best API』，最終我們重新回歸原始的 assert 作為默認的斷言庫。

簡單地說，它的優點是：

- 沒有 API 就是最好的 API，不需要任何記憶，只需 assert 即可。
- **強大的錯誤信息反饋**

報錯信息太妹太詳細，讓人有想看錯誤報告的慾望：

![](https://cloud.githubusercontent.com/assets/227713/20919940/19e83de8-bbd9-11e6-8951-bf4a332f9b5a.png)

## 測試約定

為讓更專注測試用例本身如何編寫，而不是耗費時間在如何運行測試腳本等輔助工作上，框架對單元測試做了一些基本約定。

### 測試目錄結構

我們約定 `test` 目錄為存放所有測試腳本的目錄，測試所使用到的 `fixtures` 和相關輔助腳本都應該放在此目錄下。

測試腳本文件統一按 `${filename}.test.js` 命名，必須以 `.test.js` 作為文件後綴。

一個應用的測試目錄示例：

```
test
├── controller
│   └── home.test.js
├── hello.test.js
└── service
    └── user.test.js

```

### 測試運行工具

統一使用 egg-bin 運行測試腳本，自動將內置的 Mocha、co-mocha、power-assert、nyc 等模組組合引入到測試腳本中，讓我們**聚焦精力在編寫測試代碼**上，而不是糾結選擇那些測試周邊工具和模組。

```json
{
  "script": {
    "test": "egg-bin test"
  }
}
```

然後就可以按標準的 `npm test` 來運行測試了。

```shell
npm test

> unittest-example@ test /Users/mk2/git/github.com/eggjs/examples/unittest
> egg-bin test

  test/hello.test.js
    ✓ should work

  1 passing (10ms)
```

## 準備測試

本文主要介紹如何編寫應用的單元測試，關於框架和插件單元測試請查看[框架開發](https://eggjs.org/zh-cn/advanced/framework.html)和[插件開發](https://eggjs.org/zh-cn/advanced/plugin.html)相關章節。

### mock

正常來說，如果要完整手寫一個 app 創建和啟動代碼，還是需要寫一段初始化腳本的，並且還需要在測試跑完後做一些清理工作，如刪除臨時文件，銷毀 app。

常常還有模擬各種網絡異常，服務訪問異常等特殊狀況。

所以我們單獨為框架抽取一個測試 mock 輔助模組：[egg-mock](https://github.com/eggjs/egg-mock)，有了它我們就可非常快速編寫一個 app 單元測試，並且還能快速創建一個 ctx 來測試它的屬性、方法和 Service 等。

### app