---
title: 账号互联
---

# 🔗 账号互联（Bridge）

三个应用共用一个机制：**检测到任意一个已登录，就能直接沿用，不用重新输 Token**。

## 背景

三个应用都需要 GitHub Token，如果每次切换都要重新粘贴一遍，很难用。

## 实现方式

每个应用在自己的 `js/bridge.js` 里声明：

```js
var APPS = [
  { key: 'facehub', origin: 'https://cool-zimo.github.io', path: '/FaceHub/', name: 'FaceHub' },
  { key: 'drive',   origin: 'https://cool-zimo.github.io', path: '/github_drive/', name: 'GitHub Drive' },
  { key: 'cangshu', origin: 'https://cool-zimo.github.io', path: '/cangshu/',      name: '仓鼠' }
];
```

登录时把 Token 写入 localStorage，键名带应用前缀。切到另一个应用时，
bridge 会去读另外两个的键，找到**已登录的那个**。

## 一个容易写错的地方

不能简单取"另一个应用"，必须找**确实已登录**的那个。

否则当 Drive 和仓鼠都没登录时，逻辑会指向一个空的应用，导致提示错位。

## 安全

Token 只在同源（`cool-zimo.github.io`）的三个路径之间共享。
不会写到任何公共存储，也不会随页面请求发出去。
