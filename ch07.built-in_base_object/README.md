# 框架內置基礎對象

本章會初步介紹一下框架中內置的一些基礎對象，包括 Koa 繼承而來的 4 個對象 (Application, Context, Request, Response) 以及框架擴展的一些對象 (Controller, Service, Helper, Config, Logger)，之後文檔會經常遇到它們。

## Application

Application 是全局應用對象，在一個應用中，只會實例化一個，它繼承自 Koa.Application，在它上面我們可以掛載一些全局的方法和對象。我們可輕鬆地在插件或應用中擴展 Application 對象

### 獲取方式

Application 對象幾乎可以在編寫應用時的任何一個地方獲取到，下面介紹幾個經常用到的獲取方式：

幾乎所有被框架 Loader 加載的文件 (Controller, Service, Schedule 等)，都可以 export 一個函數，這個函數會被 Loader 調用，並使用 app 作為參數。

- 啟動自定義腳本
- Controller 文件
