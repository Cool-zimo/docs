# 架构手术报告 · 桌面版退成纯桥

**日期**：2026-09-26
**版本**：gdpy v0.0.11 / github_drive（两处 js 修复）

---

## ★ 我上一轮的方案前提是错的

上一轮我说：

> 现在 Python 里有 vfs.py / transfer.py / config_sync.py，
> 前端 js 里有对应的，**同一套逻辑两份实现**

核查后的真相：

```
webview_main.py 只 import 了 Bridge 和 server.start
—— 完全不碰 vfs / transfer / share / config_sync
```

**不是"同一入口双实现"，是"两个入口各自实现，其中一个已废弃"。**

- webview 版（0.0.4 起）：业务逻辑 100% 在前端 js
- Tkinter 版（`main.py`）：用 Python 业务层，**但已不构建**

所以 Python 业务层在 webview 版里是**死代码**，
还被 `--hidden-import` 强行打进 exe 白占体积。

> **架构早就已经是"纯桥"了。我要做的不是改造，是清扫。**

---

## 删除（1450 行）

| 文件 | 行数 | 理由 |
|---|---|---|
| `main.py` | 33 | Tkinter 入口，已不构建 |
| `gdrive/ui/app.py` | 477 | Tkinter 界面 |
| `gdrive/ui/plugins_ui.py` | 122 | Tkinter 插件界面 |
| `gdrive/core/vfs.py` | 296 | 前端 vfs.js 承担 |
| `gdrive/core/transfer.py` | 273 | 前端 file-manager.js 承担 |
| `gdrive/core/share.py` | 249 | 前端 share.js 承担 |
| workflow hidden-import | — | 对应移除 |

## 保留并重新定位

| 模块 | 新定位 |
|---|---|
| `config_sync.py` | **维护工具专用**的「读-改-写」写入层。应用运行时完全不碰 config.json |
| `maintain.py` / `recover.py` | 桌面独有价值，前端 js 无对应实现 |
| `api.py` / `config.py` / `plugins.py` / `webview/*` | 纯桥 |

## 抢救：三个纯函数不该跟着埋掉

`human_size` / `breadcrumb_segments` / `shorten` 移到新模块
`gdrive/core/textutil.py` —— 不依赖 VFS 数据结构，
且与前端存在**显示契约**（格式必须一致）。

---

## 手术暴露的真 bug：js 侧缺保护

Python 侧 v0.0.9 修过「超大分块不无限 `create_repo`」，
但 **js 的 `autoSelectRepo` 从来没修**——而线上实际在跑的是 js。

```javascript
if (config.autoCreateRepo) {
    return await this.autoCreateStorageRepo();   // ← 无上限校验
}
```

后果：`canRepoFit` 永远 false → 每个分片都 `create_repo`。
500MB × 512KB 分片 = **上千次 create_repo**，
而新建的仓库同样装不下——问题没解决只是被放大。

**已在 js 侧加上限校验，js 测试 6 项。**

> 这是双实现最隐蔽的代价：我在 Python 侧修了 bug，以为修好了，
> 实际用户跑的那份从来没修。**修 bug 前要先确认修的是不是上线那份。**

---

## 手术中自己写出的 bug（实跑才暴露）

`tools/maintain_cli.py` 用了 `scan()` 的
`file_count` / `recorded_bytes` / `actual_bytes`——
**`scan()` 根本没返回这些键**。

表现：CLI 显示「记录文件 0」，实际有 59 个。

修在源头（`maintain.py` 补统计字段）+ 2 项防复发断言。

> 跟 V0.0.4 的 `idm[1]` 同类：**写了调用却没核对返回结构。**
> 如果没实跑 CLI，这个 bug 会跟着发布。

---

## 新增 `tools/maintain_cli.py`

```
python tools/maintain_cli.py scan      # 体检
python tools/maintain_cli.py recover   # 找回孤儿
```

**为什么不做成界面按钮**：`web/` 是从 github_drive 同步来的
（`tools/sync_web.py`），直接改会被下次同步覆盖。
界面入口必须提交到 github_drive（web 端），不是这里。

CLI 保证能力不丢失，且不受同步影响。

### 实跑结果

```
记录文件     : 59
记录占用     : 133.3 MB
实际占用     : 133.3 MB
★ 孤儿      : 1 个 0 B（.gitkeep，跳过）
★ 幽灵      : 0 个
```

幽灵为 0 —— 说明恢复的 44 个文件 chunks 都真实存在，恢复有效。

---

## ⚠️ 发现的新问题：记账全是 0

```
drive-storage-2026-08-30-8696  记账 0 B   实际 77.1 MB
drive-storage-2026-08-27-c4aa  记账 0 B   实际 40.9 MB
drive-storage-2026-08-25-ft3j  记账 0 B   实际 14.2 MB
drive-storage-2026-08-25-hu15  记账 0 B   实际  1.1 MB
```

`repoUsage` 完全没写。后果：`canRepoFit` 认为所有仓库都是空的，
一直往里写直到 API 返回超限才暴露。

不紧急（自动建仓会兜底），但会让"仓库快满"预警失效。

---

## 待办

| 项 | 说明 |
|---|---|
| 界面入口 | `scan` / `recover` 加到 web 端 UI（要提交到 github_drive） |
| 面包屑 | 前端 js 无面包屑。Python 参考实现已移到 `textutil.py`，需在 js 实现 |
| 记账写入 | `repoUsage` 全 0，需排查为何没写 |
| js 回滚验证 | `subtractFromRepoUsage` 有 3 处调用，但回滚正确性未验证 |
| `_recovered` 44 个文件 | 等你确认留哪些、重名的留哪个版本 |
