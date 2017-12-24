# Service

Service 就是在複雜業務邏輯下做業務邏輯封裝的一個抽象層，提供這個抽象好處：

- Controller 邏輯簡潔
- 保持業務獨立性與復用
- 容易編寫測試，可查看[這裡](https://eggjs.org/zh-cn/core/unittest.html)

## 使用場景

- 複雜資料處理
- 第三方服務的調用

## 定義 Service

```javascript
// app/service/user.js
const Service = require('egg').Service;

class UserService extends Service {
  async find(uid) {
    const user = await this.ctx.db.query('select * from user where uid = ?', uid);
    return user;
  }
}

module.exports = UserService;
```

### 屬性

每一次用戶請求，框架都會實例化對應的 Service 實例，由於它繼承 egg.Service，故有下列屬性方便開發：

- this.ctx
- this.app
- this.service
- this.config
- this.logger

### Service ctx 詳解

比如我們可用：

- this.ctx.curl 發請網路調用
- this.ctx.service.otherService
- this.ctx.db

### 注意事項

- Service 必須放在 `app/service` 目錄，可支持多級目錄。

```
app/service/biz/user.js => ctx.service.biz.user
app/service/sync_user.js => ctx.service.syncUser
app/service/HackerNews.js => ctx.service.hackerNews
```

- 一個 Service 只能包含一個類，這個類需透過 module.exports 方式返回
- Service 需透過 Class 方式定義，父類必須是 egg.Service
- Service 不是單例，是 **請求級別* 的對象，框架在每次請求中首次訪問 `ctx.service.xx` 時延遲實例化，所以 Service 中可透過 `this.ctx` 獲取到當前請求的上下文。

## 使用 Service

```javascript
// app/router.js
module.exports = app => {
  app.router.get('/user/:id', app.controller.user.info);
};

// app/controller/user.js
const Controller = require('egg').Controller;
class UserController extends Controller {
  async info() {
    const userId = ctx.params.id;
    const userInfo = await ctx.service.user.find(userId);
    ctx.body = userInfo;
  }
}
module.exports = UserController;

// app/service/user.js
const Service = require('egg').Service;
class UserService extends Service {
  // 默认不需要提供构造函数。
  // constructor(ctx) {
  //   super(ctx); 如果需要在构造函数做一些处理，一定要有这句话，才能保证后面 `this.ctx`的使用。
  //   // 就可以直接通过 this.ctx 获取 ctx 了
  //   // 还可以直接通过 this.app 获取 app 了
  // }
  async find(uid) {
    // 假如 我们拿到用户 id 从数据库获取用户详细信息
    const user = await this.ctx.db.query('select * from user where uid = ?', uid);

    // 假定这里还有一些复杂的计算，然后返回需要的信息。
    const picture = await this.getPicture(uid);

    return {
      name: user.user_name,
      age: user.age,
      picture,
    };
  }

  async getPicture(uid) {
    const result = await this.ctx.curl(`http://photoserver/uid=${uid}`, { dataType: 'json' });
    return result.data;
  }
}
module.exports = UserService;

// curl http://127.0.0.1:7001/user/1234
```