---
title: GitHub Drive 模块手册
---

# 📁 GitHub Drive · 重要模块手册

- 仓库：`Cool-zimo/github_drive`
- 代码量：9843 行（不含测试）

---

## 模块划分

| 文件 | 行数 | 职责 |
|---|---|---|
| `app.js` | 1911 | 主应用 |
| `ui.js` | 2793 | 界面渲染 |
| `storage.js` | 757 | 本地存储 + VFS |
| `share.js` | 725 | 分享 |
| `github-api.js` | 604 | REST 封装 |
| `file-manager.js` | 546 | 上传/下载/分片 |
| `bridge.js` | 577 | 跨应用互认 |
| `i18n.js` | 836 | 多语言 |
| `cangshu-link.js` | 330 | 与仓鼠联动 |
| `config-sync.js` | 233 | 配置同步 |
| `extension.js` | 289 | 扩展名识别 |

---

## 1. storage.js — 虚拟文件系统（VFS）

### 结构

```js
/drive_home/               ← 根目录，用户可见
├── 文件：{ chunks: [...] }  ← 记录分片在各仓库的实际位置
└── 文件夹：纯虚拟概念，只存在于配置中
```

**文件夹是虚拟的**——GitHub 里没有目录，只有扁平的文件名。所以文件夹只存在于 VFS 配置里，删文件夹不删任何 GitHub 文件。

### 分片配置

```js
repoNamePrefix: 'drive-storage',
chunkSize: 512 * 1024,      // 512KB
```

⚠️ **注意这个值和 FaceHub 的 2MB 不一样**。Drive 的分片更小，因为它的场景是大量小文件、需要更细的重试粒度。

**有配置迁移逻辑**：旧版本是 50MB / 20MB / 5MB，自动更新为 512KB。

```js
if (saved.chunkSize === 50*1024*1024 || saved.chunkSize === 20*1024*1024
    || saved.chunkSize === 5*1024*1024) {
    saved.chunkSize = 512 * 1024;
}
```

改这个常量时必须同步更新迁移分支，否则老用户配置不生效。

### 多账号隔离

```js
accountScopedKeys = [
  VFS, FAVORITES, RECENT, SHARES, REPOS, REPO_USAGE, STORAGE_CONFIG, SETTINGS
]
```

这些 key 按账号 ID 分开存。切账号时数据不串。

⚠️ **新增需要隔离的数据时，必须加进这个数组**，否则会串号。

---

## 2. file-manager.js — 上传与分片

### 上传流程

```
文件 → 按 chunkSize 切片
    → 逐片 uploadLargeFile / createOrUpdateFileBinary
    → 失败则清理已上传分片
```

**失败清理很重要**：不清理会在仓库里堆积垃圾分片，占容量且无法自动回收。

```js
// 上传失败，清理已上传的分片，避免垃圾文件堆积
await this.api.deleteFile(chunk.owner, chunk.repo, chunk.path, ...)
```

### 并发约束

与 FaceHub 同理：**写入必须串行**。commit 层面的 409 冲突与路径无关。

---

## 3. 回收站设计

```
移到回收站 → 只改 VFS 里的虚拟路径，不动 GitHub 分片
永久删除   → 才真正删 GitHub 分片
```

**为什么**：删除分片是 N 次 API 调用，慢且费配额。回收站只改一行配置，秒完成。

---

## 4. share.js — 分享

生成 `gd-share-*` 仓库，里面放分享元数据。拿到链接的人不需要 Token 就能下载。

---

## 5. 已知限制

| 限制 | 值 | 来源 |
|---|---|---|
| 单文件（contents API） | ~30MB 实测 | 40MB 会 422 |
| 单仓库建议 | 不超过 1GB | GitHub 软限制 |
| API 限速 | 5000 次/小时 | core |

---

## 6. 一个目录值得注意

```
repos/github_drive/Users/feng/Desktop/github_drive/
```

这是误提交进来的嵌套目录（把整个项目又塞了一份进去）。
不影响运行，但让仓库体积翻倍。可以考虑清理。
