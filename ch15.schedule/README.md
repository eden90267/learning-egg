# 定時任務

雖然我們透過框架開發的 HTTP Server 是請求響應模型的，但仍然還會有許多場景需要執行一些定時任務，例如：

1. 定時上報應用狀態
2. 定時從遠程接口更新本地緩存
3. 定時進行文件切割、臨時文件刪除

框架提供一套機制來讓定時任務的編寫和維護更加優雅。

## 編寫定時任務

所有定時任務都統一存放在 `app/schedule` 目錄下，每一個文件都是一個獨立的定時任務，可以配置定時任務的屬性和要執行的方法。

一個例子，我們定義一個更新遠程資料到內存緩存的定時任務，就可以在 `app/schedule` 目錄下創建一個 `update_cache.js` 文件。

```javascript
const Subscription = require('egg').Subscription;

class UpdateCache extends Subscription {
  // 透過 schedule 屬性來設置定時任務的執行間隔等設置
  static get schedule() {
    return {
      interval: '1m', // 一分鐘間隔
      type: 'all'
    };
  }
  
  // subscribe 是真正定時任務執行時被運行的函數
  async subscribe() {
    const res = await this.ctx.curl('http://www.api.com/cache', {
      dataType: 'json'
    });
    this.ctx.app.cache = res.data;
  }
}

module.exports = UpdateCache;
```

還可簡寫為：

```javascript
module.exports = {
  schedule: {
    interval: '1m',
    type: 'all',
  },
  async task(ctx) {
    const res = await ctx.curl('http://www.api.com/cache', {
      dataType: 'json'
    });
    ctx.app.cache = res.data;
  },
}
```

這個任務會在每一個 Worker 進程上每 1 分鐘執行一次，將遠程數據請求回來掛載到 `app.cache` 上。

### 任務

- `task` 或 `subscribe` 同時支持 `generator` 和 `async function`。
- `task` 的入參為 `ctx`，匿名的 Context 實例，可透過它調用 `service` 等。


### 定時方式

定時任務可以指定 interval 或 cron 兩種不同的定時方式。

#### interval

透過 `schedule.interval` 參數來配置定時任務的執行時機，定時任務將會每間隔指定的時間執行一次。interval 可以配製成

- 數字類型：單位為毫秒數
- 字符類型：會透過 ms 轉換成毫秒數，例如 5s

```javascript
module.exports = {
  schedule: {
    // 每 10 秒執行一次
    interval: '10s'
  }
}
```

#### cron

透過 schedule.cron 參數來配置定時任務的執行時機，定時任務將會按照 cron 表達式在特定時間點執行，cron 表達式透過 [cron-parser](https://github.com/harrisiirak/cron-parser) 進行解析。

**注意：cron-parser 支持可選的秒 (linux crontab 不支持)**

```
*    *    *    *    *    *
┬    ┬    ┬    ┬    ┬    ┬
│    │    │    │    │    |
│    │    │    │    │    └ day of week (0 - 7) (0 or 7 is Sun)
│    │    │    │    └───── month (1 - 12)
│    │    │    └────────── day of month (1 - 31)
│    │    └─────────────── hour (0 - 23)
│    └──────────────────── minute (0 - 59)
└───────────────────────── second (0 - 59, optional)
```

```javascript
module.exports = {
  schedule: {
    // 每三小時準點執行一次
    cron: '0 0 */3 * * *'
  }
}
```

### 類型

框架提供的定時任務默認支持兩種類型，worker 和 all。worker 和 all 都支持上面的兩種定時方式，只是當到執行時機時，會執行定時任務的 worker 不同：

- worker 類型：每台機器上只會有一個 worker 會執行這個定時任務，每次執行定時任務的 worker 的選擇是隨機的
- all 類型：每台機器上的每個 worker 都會執行這個定時任務

### 其他參數

- cronOptions：配置 cron 的時區等，參見 [cron-parser](https://github.com/harrisiirak/cron-parser#options) 文檔
- immediate：配置了該參數為 true 時，這個定時任務會在應用啟動並 ready 後立刻執行一次這個定時任務。
- disable：配置該參數為 true，這個定時任務不會被啟動

### 動態配置定時任務

有時我們需要判斷不同的環境來配置定時任務的參數，定時任務還有另一種寫法：

```javascript
module.exports = app => {
  return {
    schedule: {
      interval: '1m',
      type: 'all',
      disable: app.config.env === 'local',
    },
    async task(ctx) {
      const res = await ctx.curl('http://www.api.com/cache', {
        contentType: 'json',
      });
      ctx.app.cache = res.data;
    }
  }
}
```

## 手動執行定時任務

我們可透過 `app.runSchedule(schedulePath)` 來運行一個定時任務。`app.runSchedule` 接受一個定時任務文件路徑 (`app/schedule` 目錄下的相對路徑或者完整的絕對路徑)，執行對應的定時任務，返回一個 Promise。

有些場景我們可能需要手動執行定時任務，例如：

- 透過手動執行定時任務可以更優雅的編寫對定時任務的單元測試

```javascript
const mm = require('egg-mock');
const assert = require('assert');

it ('should schedule work fine', async () => {
  const app = mm.app();
  await app.ready();
  await app.runSchedule('update_cache');
  assert(app.cache);
})
```

- 應用啟動時，手動執行定時任務進行系統初始化，等初始化完畢後再啟動應用。參見[應用啟動自定義](https://eggjs.org/zh-cn/basics/app-start.html)章節，我們可在 `app.js` 中編寫初始化邏輯。

```javascript
module.exports = app => {
  app.beforeStart(async () => {
    // 保證應用啟動監聽端口前數據已經準備好了
    // 後續數據的更新由定時任務自動觸發
    await app.runSchedule('update_cache');
  });
};
```

## 擴展定時任務類型

默認框架提供的定時任務只支持每台機器的單個進程執行和全部進程執行，有些情況下，我們的服務並不是單機部署的，這時候可能有一個集群的某一個進程執行一個定時任務的需求。

框架並沒有直接提供此功能，但開發者可以在上層框架自行擴展新的定時任務類型。

在 `agent.js` 中繼承 `agent.ScheduleStrategy`，然後透過 `agent.schedule.use()` 註冊即可：

```javascript
module.exports = agent => {
  class ClusterStrategy extends agent.ScheduleStrategy {
    start() {
      // 訂閱其他的分布式調度服務發送的消息，收到消息後讓一個進程執行定時任務
      // 用戶在定時任務的 schedule 配置中來配置分布式調度的場景 (scene)
      agent.mq.subscribe(schedule.scene, () => this.sendOne());
    }
  }
  agent.schedule.use('cluster', ClusterStrategy);
}
```

ScheduleStrategy 基類提供了：

- `schedule` - 定時任務的屬性，disable 是默認統一支持的，其他配置可以自行解析
- `this.sendOne()` - 隨機通知一個 workder 執行 task
- `this.sendAll` - 通知所有的 worker 執行 task
