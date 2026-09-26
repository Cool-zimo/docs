# fd v2 执行报告 · GitHub Drive（修订版）

**日期**：2026-09-26
**方法**：fd 检查法 v2
**版本**：gdpy v0.0.10 / github_drive（web 端补丁）

---

## ★ 最重要的结论：上一版报告的判断是错的

v0.0.9 报告写：

> 孤儿数据 56.2 MB（占 42%），建议清理

**方向完全反了。那不是垃圾，是用户丢失的文件。**

### 推翻依据

看文件名：

```
义务教育教科书 · 数学六年级上册.pdf              18.3 MB
（备份）义务教育教科书·语文六年级下册_加水印.pdf    13.7 MB
quark.exe                                        14.0 MB
python.exe                                        0.2 MB
20260513-184919.jpg …                             一批照片
```

**决定性证据**：每个 blob 都是所在目录里**唯一的文件、形态完整**。
真正的删除残留应该是「36 片里剩 1 片」这种残缺状态。

| | 数量 |
|---|---|
| VFS 里能看到 | 15 个 |
| 仓库里躺着、界面看不到 | **45 个** |

**丢的比留的多 3 倍。**

### 如果执行了会怎样

`plan_purge` 若真执行删除，**45 个文件永久丢失**，含两个教科书 PDF。

**`plan_purge` 只出计划不删——这个保守设计救了场。**
但当时的判断依据（"孤儿 = 删除残留 = 可清理"）是错的。

### 教训

> **判定"这是垃圾"需要的证据强度，远高于判定"这是数据"。**
> 我用了"VFS 无记录"这一条就下了"垃圾"的结论，
> 而没有去看**文件名**和**文件完整度**这两个更直接的证据。

已写进 fd-check.md v2：孤儿必须做**路径特征分析**才能定性，
不允许只看"有无 VFS 记录"。

---

## 根因（已定位并修复）

每次上传都新建**随机目录** `mtrand/filename`，VFS 只指向最新那份，
**旧目录的 blob 从不删除**。

证据：`save.json` 在仓库里有 4 份（4 个不同随机目录）、
`math_history.json` 有 3 份——就是这么一份份累积出来的。

`uploadFile` 里上传失败会清理分片，
但**上传成功覆盖同名文件时从不清理**。

---

## 修复 1：web 端（堵源头）

`js/file-manager.js`

```javascript
// 上传开始时记住旧记录
const oldFile = this.storage.getFile(virtualPath);
const oldChunks = (oldFile && Array.isArray(oldFile.chunks))
    ? oldFile.chunks.slice() : [];

// ... 上传、写入新 VFS ...

const fileInfo = this.storage.putFile(virtualPath, {...});

// ★ 新版本已写入，此时旧分片才真正成为孤儿
if (oldChunks.length) {
    await this._cleanupOrphanChunks(oldChunks, virtualPath);
}
```

**顺序不能反**：先删旧的、新上传又失败 = 文件彻底没了。

新增 `_cleanupOrphanChunks` / `_pendingOrphan`。
`deleteFile` 里原有的 `console.warn` 改为记进 `pendingOrphanChunks`。

**关键设计：清理失败不能抛异常。**
调用方 `uploadFile` 已经成功了，因为清理旧分片失败就报"上传失败"是错的。
失败的分片留痕，可被后续扫描/重试发现。

**测试 10 项**，其中 4 条是源码级断言（防止补丁被后续改动悄悄移除）。

---

## 修复 2：desktop 端（找回数据）

新增 `gdrive/core/recover.py`，三条铁律：

1. **只增不删** —— 绝不删除任何 blob，恢复失败也不会让情况变糟
2. **落独立目录** —— 默认 `/drive_home/_recovered`，不覆盖现有文件
3. **先出计划** —— 生成补丁供确认，确认后才写回

**为什么落独立目录**：孤儿里有 12 个与现有文件重名
（`save.json` x4、`math_history.json` x3、一批 jpg），
无法判断哪个版本是用户想要的。原位覆盖会丢数据。

`maintain.py` 补收 tree item 的 **sha** —— 恢复进 VFS 后要靠它下载/删除，
缺 sha 会导致后续删除失败。

### 线上已执行

执行前已备份 config.json。

```
VFS 文件数   15 → 59
恢复文件     44 个，56.2 MB
冲突跳过     0
被忽略       1（.gitkeep）
```

---

## 其余修复（v0.0.9）

| # | 问题 | 修改 | 防复发断言 |
|---|---|---|---|
| 1 | `list_blobs` 吞异常 | 不吞，抛给 scan 记 errors | `test_unscannable_repo_not_ghost` |
| 2 | minChunkSize 跨端差 20 倍 | 默认改 10MB（线上实测值） | `test_core`【6】 |
| 3 | desktop 写回 storageConfig | `READONLY_CONFIG_KEYS`，raise | `TestReadOnly` 3 条 |
| 4 | `_FALLBACK` 落后 | 同步版本号 | `test_core`【18】3 条 |
| 5 | `sys.exit` 无 `__main__` 保护 | 包进保护里 | 【18】确实会跑（77→80） |

### 第 2 条有个 trap 值得记

`js/storage.js` 的 `getStorageConfig()` 默认值也是 512KB。
**照抄"与 js 保持一致"会抄到一个错的值**——线上真实值是 10MB，
那是用户保存设置后覆盖上去的。

> **要查线上实际值，不是代码默认值。**

### 第 5 条是最难发现的

`tests/test_core.py` 顶层 `sys.exit` 无 `__main__` 保护。
我把新测试追加到末尾，跑了显示"77 通过"——**一条都没执行**。

测试数量不变、全绿、无报错。**只有对比"加了 2 条为何总数没变"才会察觉。**

---

## 上一版报告的另一个错误（已修正）

v0.0.9 报告写「`subtractFromRepoUsage` 零调用」——**错了**。
实际有 3 处调用（`storage.js:455/479`、`file-manager.js:201`）。

原因：只 grep 了 `addToRepoUsage`，没 grep 反操作就下结论。

**fd v2 已加硬规则**：凡查"有没有做 X"，必须同时查 X 的反操作。
**对称操作只查一半 = 没查。**

---

## 测试

| 套件 | 项数 |
|---|---|
| test_core | 80 |
| test_config_sync | 29 |
| test_plugins | 135 |
| test_security | 63 |
| test_url | 16 |
| test_webview | 60 |
| test_maintain | 22 |
| test_recover | 12 |
| **合计** | **417**（+ web 端 10 项 js） |

`tools/check_resources.py` 通过。

---

## 未完成 / 待确认

| 项 | 说明 |
|---|---|
| **44 个恢复文件需要你过目** | 在 `/drive_home/_recovered`。其中 12 个与现有文件重名，需人工判断留哪个；确认后可移回原位 |
| 敏感内容提示 | 恢复出的文件里含密码截图、API key 文本、私人视频。仓库是私有的，但建议确认这些是否还要留着 |
| `pendingOrphanChunks` 无 UI | 清理失败的分片只记在内存里，刷新即丢。需要持久化和重试入口 |
| desktop 架构（Python 退成纯桥） | 本轮未做。双实现漂移的病根还在 |
