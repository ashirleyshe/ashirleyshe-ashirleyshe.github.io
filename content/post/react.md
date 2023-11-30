---
title: "React 筆記"
subtitle:    ""
description: ""
date: 2022-10-04
draft: true
author:      "Ashley Hsu"
image:       ""
tags:        ["react"]
categories:  ["Tech" ]
---

## 安裝
1. 安裝 node
2. [建立全新的 React 應用程式](https://zh-hant.reactjs.org/docs/create-a-new-react-app.html)

### Create React App
```bash
npx create-react-app my-app
cd my-app
npm start
```
### Run 
```
npm run start
```
webpack -> 打包

## JSX
利用寫 html 的方式寫 react 的物件

寫 class -> className ，因為 javasscript 已有 class

### 自動補 html tag
不用安裝 extension，vs code 右下角選語言：Javascrpit React


## style component
可以在 JSX 中撰寫 CSS code

## 流程
1. 大概分一下資料夾（pages/Home...）
2. 切版，切各個不同的元件

Home 存 states，透過 props 傳給孩子 


## useState
變數可以跟渲染機制綁在一起，就不會有變數變了畫面上沒動的問題

`setA` 用來設定 `a` 的值
```javascript
const [a, setA] = useState(100)
function plus() {
    let b = 100 + 200
    setA(b)
}
```

## 箭頭函數
