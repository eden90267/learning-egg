# 本地開發

為提升研發體驗，我們提供了便捷方式在本地進行開發、調試、單元測試等。

這裡使用到 egg-bin 模組 (只在本地開發和單元測試使用)。

```shell
npm i egg-bin --save-dev
```

## 啟動應用

本地啟動應用進行開發活動，當我們修改代碼並保存，應用會自動重啟實時生效。

### 添加命令

```json
{
  "scripts": {
    "dev": "egg-bin dev"
  }
}
```

這樣就可透過 `npm run dev` 命令啟動應用。

### 環境配置

本地啟動應用以 `env:local` 啟動，讀取的配置是 `config.default.js` 和 `config.local.js` 合併的結果。

### 指定端口

本地啟動應用默認 7001，可指定其他端口：

```json
{
  "scripts": {
    "dev": "egg-bin dev --port 7001"
  }
}
```

## 單元測試

### 添加命令

```json
{
  "scripts": {
    "test": "egg-bin test"
  }
}
```

這樣就可透過 `npm test` 命令進行單元測試。

### 環境配置

測試用例執行時，應用是以 `env:unittest` 啟動的，讀取的配置也是 `config.default.js` 和 `config.unittest.js` 合併的結果。

### 運行特定用例文件

運行 `npm test` 時會自動執行 test 目錄下的以 `.test.js` 結尾的文件 (默認 [glob](https://www.npmjs.com/package/glob) 匹配規則 `test/**/*.test.js`)。

若要單獨執行：

```shell
TESTS=test/x.test.js npm test
```

支持 [glob](https://www.npmjs.com/package/glob) 規則

### 指定 reporter

Mocha 支持多種形式的 reporter，默認使用 `spec` reporter。

可手動設置 `TEST_REPORTER` 環境變數來指定 reporter，例如使用 `dot`：

```shell
TEST_REPORTER=dot npm test
```

### 指定用例超時時間

默認 30 秒，也可手動指定 (單位毫秒)：

```shell
TEST_TIMEOUT=5000 npm test
```

### 透過 argv 方式傳參

`egg-bin test` 除了環境變量方式，也支持直接傳參，支持  mocha 的所有參數，參見：[mocha usage](https://mochajs.org/#usage)

```shell
# 傳遞參數需額外加一個 `--`
npm test -- --help

npm test "test/**/test.js"

npm test -- --reporter=dot

npm test -- -t 3000 --grep="should GET"
```

## 代碼覆蓋率

egg-bin 已經內置了 nyc 來支持單元測試自動生成代碼覆蓋率報告。

