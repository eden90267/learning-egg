# 漸進式開發

在 Egg 裡面，有插件，也有框架，前者還包含了 path 和 package 兩種加載模式，那該如何選擇？

這裡以實例方式，一步步演示，如何漸進式進行代碼演進。

全部示例代碼可參見：[https://github.com/eden90267/egg-progressive](https://github.com/eden90267/egg-progressive)

## 最初始狀態

## 插件的雛形

```
example-app
├── app
│   └── router.js
├── config
│   └── plugin.js
├── lib
│   └── plugin
│       └── egg-ua
│           ├── app
│           │   └── extend
│           │       └── context.js
│           └── package.json
├── test
│   └── index.test.js
└── package.json
```

## 抽成獨立插件

```
egg-ua
├── app
│   └── extend
│       └── context.js
├── test
│   ├── fixtures
│   │   └── test-app
│   │       ├── app
│   │       │   └── router.js
│   │       └── package.json
│   └── ua.test.js
└── package.json
```

[https://github.com/eden90267/egg-ua](https://github.com/eden90267/egg-ua)

## 沈澱到框架

```
example-framework
├── config
│   ├── config.default.js
│   └── plugin.js
├── lib
│   ├── agent.js
│   └── application.js
├── test
│   ├── fixtures
│   │   └── test-app
│   └── framework.test.js
├── README.md
├── index.js
└── package.json
```

[https://github.com/eden90267/egg-framework](https://github.com/eden90267/egg-framework)

## 寫在最後

這裡可以看到，Egg 如何一步步漸進的去進行框架演進，得益於 Egg 強大的插件機制，代碼的共建，複用和下沉，竟然可以這麼的無痛。

- 一般來說，當應用中有可能會複用的代碼，直接放到 `lib/plugin` 目錄去，如例子中的 `egg-ua`
- 當該插件功能穩定後，即可獨立出來作為一個 `node module`
- 如此以往，應用中相對複用性較強的代碼都會逐漸獨立為單獨的插件
- 當你的應用逐漸進化到針對某類業務場景的解決方案時，將其抽象為獨立的 framework 進行發布
- 當在新項目中抽象出的插件，下沈集成到框架後，其他框架只需要簡單的重新 `npm install` 下就可以使用上，對整個團隊的效率有極大的提升
- 注意：不管是 應用/插件/框架，都必須編寫單元測試，並盡量實現 100% 覆蓋率

---

參考：[https://eggjs.org/zh-cn/tutorials/progressive.html](https://eggjs.org/zh-cn/tutorials/progressive.html)