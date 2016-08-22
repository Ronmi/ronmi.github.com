+++
categories = ["typescript"]
date = "2016-07-28T11:33:02+08:00"
tags = ["typescript", "karma", "webpack", "test"]
title = "在 karma+webpack 的環境下讓 test coverage 正確顯示 typescript"
+++

可能不是最佳解，但至少 works for me

## 先備知識

至少要先做到 `karma start` 可以跑測試。

## 概念

1. `tsc` 會產生 `typescript <=> javascript` 對映的 source map
2. karma + webpack + karma-coverage 可以正確處理 js 檔

所以乾脆就用 `tsc` 把程式編成 javascript 再測，然後用 `remap-istanbul` 把最後的 test coverage 結果用 `tsc` 產生的 source map 轉回 tpescript。

## 步驟

### 準備必要的套件

`npm i -D karma-coverage karma-remap-istanbul istanbul-instrumenter-loader`

### 準備 tsc 的設定

`outDir` 可以使用，但如果你的程式是 js/ts 混合的話要考慮一下，會增加等下設定 karma 和 webpack 的困擾。

最重要的是 `sourceMap` 一定要是 true，沒有 source map 的話 `remap-istanbul` 也無用武之地。

### 設定 karma

每個設定的作用可以自己去看 [karma 文件](https://karma-runner.github.io/1.0/config/configuration-file.html)，這裡不多說明，只提我有踩到雷的地方。

假設原始碼放在 `./src`，測試碼放在 `./test` 且檔名是 `xxxxTest.ts` 的話。首先是 `files`

```js
// 在 tsc 設定 outDir: "./build" 的情況
files: [
  './build/test/**/*Test.js'
],

// 沒有開 outDir
files: [
  './test/**/*Test.js'
],
```

這邊要你自己參考跑測試時的設定，把 typescript 檔改成 javascript 檔的路徑

再來設定 `preprocessor`

```js
// 記得照上面的提示修改路徑
preprocessers: {
  './build/src/**/*.js': ['webpack', 'sourcemap', 'coverage'],
  './build/test/**/*.js': ['webpack'],
},
```

注意一下 `sourcemap` 和 `coverage` 都只要加在原始碼的部份就好了

然後是 `webpack`

```js
webpack: {
  devtool: 'source-map',
  resolve: {
    // 記得不要加 .ts 和 .tsx
    extensions: ["", ".webpack.js", ".web.js", ".js", ".json"],
  },
  module: {
    loaders: [
      // 用 enzyme 必加，詳見 https://github.com/airbnb/enzyme/issues/47
      { test: /\.json$/, loader: 'json' },
      // 底下加上其他你測試有用到的 loader (用來 load typescript 的就不用加了)
      {
        test: /\.css/,
        loader: "style-loader!css-loader",
        exclude: /css\//,
      },
    ],
    preLoaders: [
      // 原始碼的 js 要用 istanbuul-instrumenter 處理一下，才會有正確的 coverage 資訊出來
      { test: /\.js$/, loader: "istanbul-instrumenter", exclude: /(node_modules|test)/ }
    ]
  },
  // 這段也是用 enzyme 才需要的資訊
  externals: {
    'react/addons': true,
    'react/lib/ExecutionEnvironment': true,
    'react/lib/ReactContext': true,
  },
},
```

接著設定 `reporters`，要加上 `coverage` 和 `karma-remap-istanbul` 和它們的設定

```js
reporters: ['mocha', 'coverage', 'karma-remap-istanbul'],

coverageReporter: {
  dir: './coverage',
  reporters: [
    { type: 'json', subdir: '.' },
  ],
},
remapIstanbulReporter: {
  src: './coverage/coverage-final.json',
  reports: {
    html: './coverage',
  },
},
```

因為 `remap-istanbul` 要資料來源是 json 檔，所以要先用 `karma-coverage` 產生 json，再用 `remapIstanbulReporter.src` 指定正確的檔案位址。

最後，不要忘了在 `plugins` 加上 `karma-coverage` 和 `karma-remap-istanbul`

## 跑看看

```sh
tsc && karma start
```

我相信你第一次跑應該是失敗的，第二次跑可能就成功了。

這是因為 `remap-istanbul` 會用 `watch` 監聽 json 檔的變化。但這個 watch 很兩光：檔案原本不存在的話就沒辦法監聽了。所以要先用 `touch` 確認 json 檔存在

```sh
mkdir -p ./coverage && touch ./coverage/coverage-final.json && tsc && karma start
```

## 所以說

不同的專案肯定會需要一些不同的設定，所以記好這個解法的概念比較重要：

    先用 tsc 編出 js 檔，然後把測試的設定裡的 ts 來源通通換成相應的 js

或是

    有錢就好了