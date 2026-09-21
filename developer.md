---
title: 开发者手册
---

# 🛠️ 开发者手册

> 本地开发、测试、发布、以及**那些踩过的坑**。

---

## 一、本地开发

### FaceHub / tiny-md

无构建、无依赖。但用了模块化写法（部分文件是 IIFE），建议起 HTTP 服务：

```bash
cd repos/FaceHub && python3 -m http.server 8000
# http://localhost:8000
```

⚠️ **不能双击 `index.html`**：`file://` 下 ES Modules 会被 CORS 拦掉，
`localStorage` 也有 origin 限制。

### 仓鼠

强制需要 HTTP（ES Modules）：

```bash
cd repos/cangshu && python3 -m http.server 8000
```

---

## 二、测试

三个应用都是 **node 跑 .mjs**，无框架、无依赖（jsdom 除外）。

```bash
node test_crypto.mjs
node test_auto_reply.mjs
node test_hl.mjs
```

### 测试里提取函数的技巧

很多模块是 IIFE 挂全局，node 里没有 DOM。做法是用正则**从源码里抠出函数体**再 `new Function`：

```js
const pats = [
  /function esc\(s\) \{[\s\S]*?\n    \}/,
  /function md\(s\) \{[\s\S]*?\n    \}/,
  ...
];
const parts = pats.map(re => (src.match(re) || [''])[0]).join('\n');
const { esc, md } = new Function(parts + '\nreturn { esc, md };')();
```

⚠️ **加新依赖时必须同步更新这张表**，否则旧测试会 `ReferenceError`。
这个已经踩过三次了（math 上线、hl 上线各一次）。

### jsdom 环境

```js
const { JSDOM } = require('jsdom');
const dom = new JSDOM(html, { url: 'https://example.com/' });
```

⚠️ **必须传 `url`**。不传的话默认是 `about:blank`，访问 `localStorage`
会抛 `SecurityError`（报错信息是空的 `DOMException {}`，极难定位）。

---

## 三、发布流程

### 1. 推代码

```python
# 用 GitHub contents API 直接 PUT（带 sha）
api('/repos/Cool-zimo/FaceHub/contents/js/app.js', 'PUT', {
    'message': '...',
    'content': base64.b64encode(data).decode(),
    'sha': sha          # 更新已存在的文件必须带
})
```

不带 `sha` 更新会 422。

### 2. 等 Pages 构建

```bash
sleep 90
curl .../repos/{owner}/{repo}/pages   # 看 status
```

⚠️ **构建中查询会返回 `errored`**——那是中间态，不是真失败。
复查一次通常就变 `built`。别看到 errored 就慌。

### 3. jsDelivr 缓存（tiny-md）

CDN 缓存最多 24 小时。想立刻生效：把 `@main` 换成具体 commit SHA。

---

## 四、踩过的坑（按严重程度排序）

### 1. ★ `hidden` 属性被 `display` 覆盖

```css
#sheet { display: flex; }   /* 作者样式 */
```

`[hidden] { display: none }` 来自**浏览器默认样式表**，优先级**低于作者样式**。
所以 `el.hidden = true` 设上了，面板照样显示——表现为"关闭按钮没反应"。

**修法**：

```css
#sheet[hidden] { display: none; }
```

用 `id + 属性` 提高特异性，不用 `!important`。

> 凡是给元素设了 `display` 又想用 `hidden` 控制显隐，都得补这条。

### 2. ★ 正则忘了捕获组

```js
var idm = /^[A-Za-z_$@][\w$]*/.exec(...)   // ← 没有括号
var w = idm[1];                              // undefined
i += w.length;                               // TypeError，整个渲染崩溃
```

**肉眼看了两遍都没发现**，是测试跑出来的第一行就炸。

### 3. ★ 裸 `fetch` 在测试里拦不住

在 `new Function('window', src)` 里写裸 `fetch`，解析的是 **Node 的全局 fetch**，
不是注入到 `window` 上的 mock——测试根本拦不住，会真的发网络请求。

**改成 `global.fetch`**，模块只依赖传入的 `global`。

### 4. 事件委托 vs 逐个绑定

面板关闭按钮**必须走事件委托**：

```js
root.addEventListener('click', function (e) {
    var t = e.target;
    while (t && t !== root) {
        if (t.classList.contains('pane-close')) { ... }
        t = t.parentNode;
    }
});
```

**为什么**：后插入的节点逐个 bind 的话一个都监听不到。事件委托天然支持动态节点。

（用 `closest()` 也行，但 `while` 向上遍历兼容性更好。）

### 5. 窄屏下 `absolute` 面板会消失

```
单列布局 → .chat-pane 变 position:fixed 脱离流
        → 父容器 #content-col 被挤成高度 0
        → .pane-full 是 absolute + inset:0，相对 0 高的父容器
        → 面板高度 0，完全看不见
```

**修法**：窄屏下 `.pane-full` 也改 `fixed`。

⚠️ **媒体查询必须放在文件末尾**——否则会被后面的
`.apps-pane { position: relative }` 盖掉（同特异性，后者胜）。

### 6. 流式渲染的节流与收尾

流式输出每个 delta 全量重渲染会掉帧。节流到 60ms 一档。

⚠️ **流结束必须 `flushRender()` 强制收尾**，否则最后一截卡在节流里不显示。

### 7. 滚动位置恢复

消息异步解密，只设一次 `scrollTop` 时 `scrollHeight` 还很矮，浏览器把你夹回 0。

**改重试式**：每 120ms 试一次，最多 20 次；用户自己滚了就立刻让位。

### 8. 分片必须串行

并发写不同路径也 409（commit 层面冲突）。**写入串行，读取并发。**

### 9. 测试环境没有 FileReader

Node 里没有浏览器 API，`uploadImage` 直接崩。测试要 mock 掉压缩和读取，
只保留上传写文件、JSON 结构那部分的真实验证。

### 10. GitHub 限流的误判

沙盒环境里 `api.cloudflare.com`、智谱 API 等全都返回
`403 "Request denied / No policy rule matched"`——**这是出口网关拦的，
不是对方的真实响应**。

**判据**：裸 GET 一个不需要鉴权的公开接口也 403，就说明是网关问题。

不要拿这种结果下"某服务不支持 X"的结论。

---

## 五、Git 历史的不可删除性

⚠️ **GitHub API 不能真正删除数据，只能"清空当前状态"。**

删掉的 comment / 文件仍在 Git 历史里，通过 commit SHA 能拿到。

对 FaceHub 的实际影响很低——消息是 E2E 密文，没有私钥就是乱码。
但**要在文档里说清楚**，不能让用户以为"删了就没了"。

**真要彻底清除**：只能删掉整个仓库重建。

---

## 六、提交前的自检清单

- [ ] `node --check` 过一遍改动的 JS
- [ ] 相关测试全绿
- [ ] 新加的模块依赖同步进测试的提取表
- [ ] 新加面板记得调 `mountIcons()` 和检查 `[hidden]` 规则
- [ ] 媒体查询放文件末尾
- [ ] 涉及写入 → 检查三个缓存 key 是否都清了

---

## 七、测试文件索引

| 文件 | 覆盖 |
|---|---|
| `test_crypto.mjs` | 加密往返、密钥派生、备份导入导出 |
| `test_auto_reply.mjs` | AI 回复防重、防循环、冷却、429 |
| `test_hl.mjs` | 代码高亮、XSS、缓存 |
| `test_md.mjs` | Markdown 渲染 |
| `test_math.mjs` | 数学公式 |
| `test_pane_narrow.mjs` | 窄屏面板可见性（CSS 规则位置） |
| `test_layout_ux.mjs` | 布局与交互 |
| `test_icons.mjs` | 图标挂载 |
