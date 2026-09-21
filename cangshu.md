---
title: 仓鼠模块手册
---

# 🐹 仓鼠 · 重要模块手册

- 仓库：`Cool-zimo/cangshu`
- 定位：**GitHub 仓库管理面板**（不是游戏）

---

## 模块划分

| 文件 | 职责 |
|---|---|
| `index.html` | 登录页 + 主界面 |
| `js/api.js` | GitHub REST 封装（含中文安全 base64） |
| `js/config.js` | 配置仓库读写 + 历史快照 |
| `js/app.js` | 主应用（卡片/文件树/仓库 CRUD），约 51KB |
| `js/bridge.js` | 跨应用登录互认 |
| `js/context-menu.js` | 右键菜单组件 |
| `js/icons.js` | 内联 SVG 图标 |

---

## 1. config.js — 配置仓库

### 为什么不放 localStorage

> 换台设备、换个浏览器就全丢了。存在仓库里才能多端同步。

这是三个应用的统一思路。

### 存储位置

```
你的账号/cangshu-config（私有，自动创建）
└── cangshu.json
```

```jsonc
{
  "version": 1,
  "updatedAt": "2026-09-11T...",
  "managed": [
    { "owner": "Cool-zimo", "repo": "xiudao",
      "note": "", "alias": "", "addedAt": "..." }
  ],
  "settings": { "configRepo": "cangshu-config", "defaultBranch": "main" }
}
```

### 构造函数支持注入仓库名

```js
new ConfigStore(api, owner, repoName)
```

**为什么**：测试用它隔离到独立仓库，否则端到端测试会直接改写用户的真实配置。

这是个值得学习的设计——**可注入 = 可测试**。

### 历史快照

每次保存前把当前（服务端）版本另存一份，改错了能回滚。

### ensureRepo 的行为

```
getRepo → 200 就用它，并读真实 default_branch
       → 404 就创建（private: true, autoInit: true）
```

⚠️ **不能硬编码 `main`**——有些账号默认分支是 `master`。必须读 `default_branch`。

---

## 2. api.js — 中文安全的 base64

```js
export function encodeBase64Utf8(str) { ... }
```

**为什么自己实现**：浏览器原生 `btoa()` 只认 Latin-1，遇到中文直接抛
`InvalidCharacterError`。

GitHub API 要求文件内容必须是 **UTF-8 的 base64**，所以这里不能图省事。

正确姿势：

```js
// TextEncoder 转 UTF-8 字节 → 逐字节转 binary string → btoa
```

---

## 3. app.js — 主应用

| 功能 | 说明 |
|---|---|
| 卡片视图 | 可见性、语言、大小、Star、分支、**Pages 状态**、**最新 commit hash** |
| 文件浏览 | 递归文件树，点击预览文本文件 |
| VS Code 集成 | 右键文件 → 在 vscode.dev 打开 |
| 仓库 CRUD | 新建（可私有/初始化 README/启用 Pages）、改名、切公开私有、删除 |

**删除是双重确认**——这个操作不可逆。

---

## 4. 权限要求

| 场景 | 权限 |
|---|---|
| 只管公开仓库 | `public_repo` |
| 要管私有仓库 | `repo`（全选） |

⚠️ **这个 token 能删你名下的仓库**，权限很大。用完建议 revoke。

---

## 5. 安全

- Token 只存 `localStorage`，只与 GitHub 官方 API 通信
- 纯静态，无后端、无埋点、无第三方统计
- **删除仓库不可逆**；只想从面板移除就选「从管理列表移除」

---

## 6. 本地开发

用了 **ES Modules**，不能直接双击 `index.html`——`file://` 协议下模块加载会被 CORS 拦掉。

```bash
python3 -m http.server 8000
# 打开 http://localhost:8000
```
