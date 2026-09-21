# 🐹 仓鼠 Cangshu

> 囤好你的每一个 GitHub 仓库 —— 一个纯前端的 GitHub 仓库管理面板。

- 在线：<https://cool-zimo.github.io/cangshu/>
- 仓库：`Cool-zimo/cangshu`
- License：MIT

---

## 它是什么

一个**没有后端的静态页面**，用 GitHub REST API 直接管理你名下的仓库。

和你平时在 github.com 上做的事一样 —— 新建、改名、切公开/私有、删仓库、
浏览文件 —— 但界面是一个统一的面板，而且能**多选批量操作**。

典型场景：账号里攒了上百个仓库（比如各种 `drive-storage-*`、`gd-share-*`
这类自动生成的散仓），一个个去网页上点太慢，用面板批量处理会快很多。

---

## 核心特性

| 功能 | 说明 |
|---|---|
| **登录** | 输入 Token（`ghp_` / `github_pat_`），自动验证并记忆账号 |
| **配置同步** | 管理列表存在**私有仓库** `cangshu-config` 里，换设备不丢 |
| **新建仓库** | 可选私有、自动初始化 README、一键启用 Pages |
| **仓库管理** | 改名、切换公开/私有、删除（双重确认） |
| **卡片视图** | 可见性、语言、大小、Star、分支、**Pages 状态**、**最新 commit hash** |
| **文件浏览** | 递归文件树，点击预览文本文件 |
| **VS Code 集成** | 右键文件 → 直接在 vscode.dev 打开编辑 |
| **右键菜单** | 卡片、文件、空白处都有上下文菜单 |

---

## 令牌需要什么权限

| 场景 | 权限 |
|---|---|
| 只管公开仓库 | `public_repo` |
| 要管私有仓库 | `repo`（全选） |

**建议**：用 fine-grained token，只勾 `Contents` + `Administration` 读写，
并把有效期设短一些。

⚠️ 这个 token 能删你名下的仓库 —— 权限很大，用完记得 revoke。

---

## 数据存在哪

### 配置文件

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

**为什么不放 localStorage**：换台设备、换个浏览器就全丢了。存在仓库里才能
多端同步 —— 和 github_drive 是一个思路。

**每次保存会先把当前版本另存一份历史快照**，改错了能回滚。

### 删掉配置仓库会怎样

只丢失"管理列表"（你登记了哪些仓库），**不影响任何被管理的仓库本身**。

---

## 安全

- Token 只存在浏览器 `localStorage`，**不会**上传到除 GitHub 官方 API
  之外的任何服务器。
- 纯静态页面，没有后端、没有埋点、没有第三方统计。
- **删除仓库不可逆。** 若只想从面板移除，右键卡片选「从管理列表移除」。

---

## 代码结构

```
cangshu/
├── index.html            登录页 + 主界面
├── css/style.css         GitHub Primer 风格
└── js/
    ├── api.js            GitHub REST 封装（含中文安全的 base64）
    ├── config.js         配置仓库读写 + 历史快照
    ├── bridge.js         跨应用登录互认（与 FaceHub / Drive）
    ├── context-menu.js   右键菜单组件
    ├── icons.js          内联 SVG 图标
    └── app.js            主应用（卡片 / 文件树 / 仓库 CRUD）
```

### 一个实现细节：中文安全的 base64

`api.js` 里自己实现了 `encodeBase64Utf8`。原因是浏览器原生的 `btoa()`
只认 Latin-1，遇到中文直接抛 `InvalidCharacterError`。

GitHub API 要求文件内容必须是 UTF-8 的 base64，所以这里不能图省事。

---

## 跨应用登录

`bridge.js` 让仓鼠和 **FaceHub**、**GitHub Drive** 互认登录状态。

三个应用共用同一套身份仓库模式，检测到另外两个里有一个已登录，
就能直接沿用，**不用重新输 Token**。

---

## 本地开发

用了 ES Modules，**不能直接双击 `index.html`**（file:// 协议下模块加载会被
CORS 拦掉），必须通过 HTTP 打开：

```bash
python3 -m http.server 8000
# 打开 http://localhost:8000
```

## 测试

```
test.mjs              主流程
test-boot.mjs         启动/登录
test-bridge.mjs       跨应用互认
test-config*.mjs      配置读写与快照
test-icons.mjs        图标挂载
test-menu.mjs         右键菜单
```
