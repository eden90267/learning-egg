# 啟動自定義

我們常需要在應用啟動期間進行一些初始化工作，等初始化完成後才可啟動成功，並開始對外提供服務。

框架提供統一入口 app.js 進行啟動過程自定義，這個文件只返回一個函數。例如，我們須在啟動期間從遠程接口加載一份全國程式列表，以便 Controller 中使用：

```javascript
// app.js
module.exports = app => {
  app.beforeStart(async() => {
    // 應用會等待這個函數執行完成才啟動
    app.cities = await app.curl('http://example.com/city.json', {
      method: 'GET',
      dataType: 'json',
    });
  })
}
```

在 Controller 就可使用：

```javascript
// app/controller/home.js
class HomeController extends Controller {
  async index() {
    // ctx.app.cities 在上面啟動期間已經加載，可以直接使用
  }
}
```

注意：在 `beforeStart` 中不建議做太耗時的操作，框架會有啟動的超時檢測。