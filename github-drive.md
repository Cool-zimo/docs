---
title: GitHub Drive 模块手册
---

# 📁 GitHub Drive · 重要模块手册

> 把 GitHub 当网盘用。所有描述对照 `js/` 下实际源码。

- 在线：<https://cool-zimo.github.io/github_drive/>
- 仓库：`Cool-zimo/github_drive`
- 版本：`v0.0.44`（`internalVersion: 44`）
- 代码量：9843 行（不含测试）

---

## 模块划分

| 文件 | 行数 | 职责 |
|---|---|---|
| `ui.js` | 2793 | 界面渲染（最大的模块） |
| `app.js` | 1911 | 主应用 + 插件市场 |
| `storage.js` | 757 | 本地存储 + **VFS** |
| `share.js` | 725 | 分享 |
| `github-api.js` | 604 | REST 封装（含 Git 底层 API） |
| `file-manager.js` | 546 | 上传/下载/分片 |
| `bridge.js` | 577 | 跨应用互认 |
| `i18n.js` | 836 | 多语言 |
| `cangshu-link.js` | 330 | 与仓鼠联动 |
| `config-sync.js` | 233 | 配置跨设备同步 |
| `extension.js` | 289 | 浏览器扩展通信 + OAuth |
| `icons.js` | 166 | 内联 SVG |

---

## 1. storage.js — 虚拟文件系统（VFS）

### 结构

```js
{
  files:   { "/drive_home/文档/a.pdf": { name, type, size, chunks, createdAt, updatedAt } },
  folders: { "/drive_home/文档":       { name, type, createdAt } }
}
```

**文件夹是纯虚拟的** —— GitHub 里没有目录概念，只有扁平文件名。
所以文件夹只存在于 VFS 配置里，**删文件夹不会删任何 GitHub 文件**。

### 路径规范化

```js
Storage.normalizePath('文档/a.pdf')  // → '/drive_home/文档/a.pdf'
```

- 强制以 `/drive_home` 开头
- 去掉结尾斜杠
- `_ensureParentFolders()` 会自动补齐父目录链

### getVFS 的防御式规范化（重要）

```js
if (!vfs || typeof vfs !== 'object') return { files: {}, folders: {} };
if (!vfs.files   || typeof vfs.files   !== 'object') vfs.files   = {};
if (!vfs.folders || typeof vfs.folders !== 'object') vfs.folders = {};
```

**为什么**：VFS 会通过 `config-sync` 从别的设备同步过来。
旧版本的数据可能是空对象或缺字段，直接 `Object.entries(vfs.files)`
会崩。这三行是跨设备同步的必备防御。

### 分片配置

```js
repoNamePrefix: 'drive-storage',
chunkSize: 512 * 1024,      // 512KB
```

⚠️ **与 FaceHub 的 2MB 不同**。Drive 场景是大量小文件，需要更细的重试粒度。

**有配置迁移逻辑**：

```js
if (saved.chunkSize === 50*1024*1024 || saved.chunkSize === 20*1024*1024
    || saved.chunkSize === 5*1024*1024) {
    saved.chunkSize = 512 * 1024;
}
```

⚠️ **改这个常量时必须同步更新迁移分支**，否则老用户配置不生效。

### 多账号隔离

```js
accountScopedKeys = [
  VFS, FAVORITES, RECENT, SHARES, REPOS, REPO_USAGE, STORAGE_CONFIG, SETTINGS
]
```

这些 key 按账号 ID 分开存（`_accountKey()` 加前缀）。切账号数据不串。

⚠️ **新增需要隔离的数据时，必须加进这个数组**，否则会串号。

### 仓库容量记账

```js
getRepoUsage()                     // { "owner/repo": { size, updatedAt } }
setRepoUsage(owner, repo, bytes)
addRepoUsage(owner, repo, bytes)
subRepoUsage(owner, repo, bytes)
```

本地记账，用来决定往哪个 `drive-storage-*` 仓库写。满了就新建。

---

## 2. github-api.js — REST 封装

**两层 API**：

| 层 | 方法 | 用途 |
|---|---|---|
| **Contents API** | `createOrUpdateFile` / `createOrUpdateFileBinary` / `deleteFile` | 单文件读写 |
| **Git 底层 API** | `createBlob` → `createTree` → `createCommit` → `updateRef` | 批量/大文件 |

### ★ batchUploadFiles 用的是 Git API，不是 Contents API

```js
const ref = await this.getRef(owner, repo, `heads/${branch}`);
const baseTreeSha = (await this.getCommit(...)).tree.sha;
// 逐个 createBlob 收集 sha
const tree = await this.createTree(owner, repo, treeItems, baseTreeSha);
const commit = await this.createCommit(owner, repo, message, tree.sha, [latestCommitSha]);
await this.updateRef(owner, repo, `heads/${branch}`, commit.sha);
```

**为什么**：N 个文件 → **1 次 commit**，而不是 N 次。
既省配额（1 次 vs N 次），也避免 N 次 commit 之间的冲突。

**代价**：必须先拿 `baseTreeSha`，并发调用时会互相覆盖（同 FaceHub 的分片串行问题）。

### getFileRawViaGit

大文件走 `/git/blobs/{sha}`（可到 100MB），绕开 Contents API 的 1MB 限制。

---

## 3. file-manager.js — 上传与分片

```
文件 → 按 chunkSize 切片
    → 逐片上传（串行）
    → 失败则清理已上传分片
```

### ★ 失败清理很重要

```js
// 上传失败，清理已上传的分片，避免垃圾文件堆积
await this.api.deleteFile(chunk.owner, chunk.repo, chunk.path, ...)
```

不清理会在仓库里堆积垃圾分片 —— 占容量且无法自动回收。

### 并发约束

**写入必须串行。** commit 层面的 409 冲突与路径无关（FaceHub 实测过：
并发 5 个不同路径 → 3 成功 2 失败）。

---

## 4. 回收站设计

```
移到回收站 → 只改 VFS 里的虚拟路径，不动 GitHub 分片
永久删除   → 才真正删 GitHub 分片
```

**为什么**：删除分片是 N 次 API 调用，慢且费配额。回收站只改一行配置，秒完成。

---

## 5. share.js — 分享

### 机制

创建**公开仓库** `gd-share-*`（或 `share-*`）+ 启用 GitHub Pages，
生成一个下载页面。拿到链接的人**不需要 Token** 就能下载。

### ★ 分享列表的真实来源是仓库，不是 localStorage

```js
async listMyShares() {
    // 拉取账号下所有分享仓库，最多翻 5 页
    if (!/^(gd-share-|share-)/i.test(repo.name)) continue;
    ...
    fromRemote: true   // 标记来源
}
```

**为什么**：分享记录原本只存 localStorage，换浏览器/清缓存/换设备就全没了，
**但仓库本身还在 GitHub 上**。

所以真实来源是**账号的仓库列表**，本地记录只用来补充描述等元信息。
这个设计和仓鼠的配置仓库是同一个思路（见[设计决策 D12](decisions.md#d12--仓鼠的配置为什么存仓库而不是-localstorage)）。

### ⚠️ 分享密码的实现有安全问题

```js
setSharePassword(shareId, password) {
    passwords[shareId] = btoa(password);     // ← base64，不是加密
}
verifySharePassword(shareId, inputPassword) {
    const saved = this.getSharePassword(shareId);
    return saved === null || saved === inputPassword;
}
```

三个问题：

1. **`btoa()` 是编码不是加密**，逆向即可还原
2. **密码只存在本地 localStorage** —— 换设备就没了，等于分享链接在别的设备上打不开
3. 校验是**纯前端**的 —— 分享仓库是公开的，别人拿到仓库内容绕过了这个校验

**这个"密码"实际上只防误点，不防真正想看的人。** 别用它保护敏感内容。

另外 `btoa()` 只认 Latin-1，**中文密码会直接抛 `InvalidCharacterError`**。

---

## 6. config-sync.js — 配置跨设备同步

```js
configRepo = 'github-drive-config';   // 私有
configFile = 'config.json';
```

`init()` 流程：确保仓库存在 → 从远程恢复 → 后续变更防抖推送（`syncTimer`）。

有 `isSyncing` 锁和 `lastSyncHash`，避免重复同步。

---

## 7. cangshu-link.js — 与仓鼠联动

让 Drive 能读写**仓鼠**的管理清单（`cangshu-config/cangshu.json`），两个应用数据互通。

### ★ 为什么不走 localStorage

> localStorage 只在同源同浏览器下有效，换个浏览器就没了。

所以直接用 Drive 自己的令牌走 GitHub API 读写仓鼠的配置仓库。

### 写操作必须先取 sha

```js
// GitHub Contents API 更新文件要求带 sha，否则 409
// 多端并发时取到旧 sha 也会 409，这里做了重试
```

---

## 8. extension.js — 浏览器扩展 + OAuth

通过 `window.postMessage` 与浏览器扩展通信（扩展可提供 Token、代理 API 请求）。

### ⚠️ OAuth clientSecret 硬编码在前端

```js
this.clientId = 'Ov23liH51YfXFWysljeU';
this.clientSecret = '20a8407220227eaac73c41e5f874f641037306d5';
```

**这是 OAuth App 的 client secret，出现在公开仓库的前端代码里。**

对公开的前端应用来说这其实**难以避免**（纯前端无法保密任何东西），
但必须知道后果：

- 任何人都能拿这个 secret 冒充你的应用
- **真正的安全边界是 redirect_uri**，GitHub 只回调白名单里的地址

**建议**：去 GitHub OAuth App 设置里确认 redirect_uri 白名单严格，
并考虑轮换这对凭据。

---

## 9. 版本管理

`js/version.js` **由 `release.py` 自动生成**：

```js
const APP_VERSION = { internalVersion: '44', formalVersion: '0.0.44', displayVersion: 'v0.0.44' };
```

不要手改。走脚本：

```bash
python release.py patch "提交信息"   # z+1
python release.py minor "提交信息"   # y+1, z归零
python release.py major "提交信息"   # x+1, y,z归零
```

脚本会同步更新 `assets/version-map.json`、`js/version.js`、`index.html`。
`version-manager.py` 负责版本切换/回滚。

---

## 10. i18n.js — 多语言

836 行，**目前只有一套语言包 `zh-CN`**。结构是完整的 key-value 映射，
加语言只需在 `translations` 下加一个 `'en-US': { ... }`。

---

## 11. 插件市场（在 app.js 里）

```js
PLUGIN_REPO = 'Cool-zimo/github_drive_plugins';
```

发现规则：

1. 仓库名含 `plugin`
2. 不是官方索引仓库本身
3. **必须有 `plugin.json`**
4. `plugin.json` 里必须指定入口文件

安装 = 拉取入口 HTML 存进 `localStorage['gd_plugins']`。

⚠️ 社区插件走 GitHub 仓库搜索，**限速 30 次/分钟**（比 core 的 5000/小时严得多）。

---

## 12. 已知限制

| 限制 | 值 | 来源 |
|---|---|---|
| 单文件（Contents API） | ~30MB 实测 | 40MB 会 422 |
| 单仓库建议 | 不超过 1GB | GitHub 软限制 |
| core API | 5000 次/小时 | |
| 搜索 API | 30 次/分钟 | 插件市场用 |

---

## 13. 仓库体积

实测（API 查 tree，2026-09）：

```
总条目      34
blob 总体积 937,727 字节（约 916 KB）
```

| 文件 | 大小 |
|---|---|
| `assets/icon-drive-512.png` | 210 KB |
| `js/app.js` | 91 KB |
| `assets/favicon.ico` | 83 KB |
| `css/style.css` | 49 KB |

**体积主要在图标资源，不是代码。** 要瘦身优先压 PNG。

> ⚠️ 早期文档写过「`Users/feng/Desktop/github_drive/` 是误提交的嵌套目录，
> 让仓库体积翻倍」—— **是错的**。实际查 tree 仓库里**完全没有 `Users/`**，
> 那只是本地工作副本里的两个 1KB 残留文件，已清理。属于没核实就写。
