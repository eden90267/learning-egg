# 目錄結構

這裡簡單瞭解下目錄約定規範：

```
egg-project
├── package.json
├── app.js (可选)
├── agent.js (可选)
├── app
|   ├── router.js
│   ├── controller
│   |   └── home.js
│   ├── service (可选)
│   |   └── user.js
│   ├── middleware (可选)
│   |   └── response_time.js
│   ├── schedule (可选)
│   |   └── my_task.js
│   ├── public (可选)
│   |   └── reset.css
│   ├── view (可选)
│   |   └── home.tpl
│   └── extend (可选)
│       ├── helper.js (可选)
│       ├── request.js (可选)
│       ├── response.js (可选)
│       ├── context.js (可选)
│       ├── application.js (可选)
│       └── agent.js (可选)
├── config
|   ├── plugin.js
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (可选)
|   ├── config.local.js (可选)
|   └── config.unittest.js (可选)
└── test
    ├── middleware
    |   └── response_time.test.js
    └── controller
        └── home.test.js
```

如上，由框架約定的目錄：

- `app/router.js`：用於配置 URL 路由規則
- `app/controller/**`：用於解析用戶的輸入，處理後返回相應的結果
- `app/service/**`：用於編寫業務邏輯層，可選，建議使用
- `app/middleware/**`：用於編寫中間件，可選
- `app/public/**`：用於放置靜態資源，可選，可參見內置插件 [egg-static](https://github.com/eggjs/egg-static)
- `app/extend/**`：用於框架的擴展，可選
- `config/config.{env}.js`：用於編寫配置文件
- `config/plugin.js`：用於配置需要加載的插件
- `test/**`：用於單元測試
- `app.js` 和 `agent.js` 用於自定義啟動時的初始化工作，可選

由內置插件約定的目錄：

- `app/public/**`：用於放置靜態資源，可選，可參見內置插件 egg-static
- `app/schedule/**`：用於定時任務，可選

**若需自定義自己的目錄規範，參見 [Loader API](https://eggjs.org/zh-cn/advanced/loader.html)**

- `app/view/**`：用於放置模板文件，可選，由模板插件約定
- `app/model/**`：用於放置領域模型，可選，由領域類相關插件約定，如 [egg-sequelize](https://github.com/eggjs/egg-sequelize)
