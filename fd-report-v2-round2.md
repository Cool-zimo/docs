# fd v2 第二轮检查报告 · desktop + web

**检查范围**：`Cool-zimo/gdpy`（桌面版）、`Cool-zimo/github_drive`（网页版）、线上数据
**方法**：读源码 → 查线上真实状态 → 冒烟测试 → 收尾检查（版本/打包/release）
**测试**：428 项（修复前 415，新增 13）
**版本**：0.0.11 → **0.0.12**

---

## 结论

| 级别 | 问题 | 状态 |
|---|---|---|
| **P0** | `vfs` 陈旧快照可被回退 → 39 个幽灵文件 | 已修 |
| **P0** | `set_exec_enabled` 暴露给 JS → 页面可自行解锁终端 | 已修 |
| P1 | 容量记账虚高 19.3 MB | 已修 |
| P2 | workflow 版本号写入取空值 | 已修（被兜底） |
| P2 | `ghproxy` 在白名单 → 可代理任意公网 URL | 已记录 |
| P2 | shim 只覆盖 fetch，XHR/Image 可外传 | 已记录 |

---

## 一、P0-1：`vfs` 陈旧快照回退（幽灵文件）

### 现象

线上 `config.json` 同时存在两个键：

```
fileIndex.files = 20   ← 当前真相
vfs.files       = 59   ← 三个月前的旧快照，其中 39 个分片已被我删除
```

### 根因

`config_sync.pull()` 的回退判据太宽：

```python
for key in ('fileIndex', 'vfs'):
    cand = cfg.get(key)
    if isinstance(cand, dict):
        vfs = cand; break      # fileIndex 取到 None 就继续看 vfs
```

只要 `fileIndex` 某次被写成空/None，就会**静默回退到那份陈旧 vfs** ——
39 个文件"看得见但下载必失败"。

这类故障比"文件消失"更难排查：**界面看起来一切正常**，只有点下载时才失败。

### 修法

回退条件从「取不到值」收紧为「键根本不存在」：

```python
if isinstance(cfg.get('fileIndex'), dict):
    vfs = cfg['fileIndex']
elif 'fileIndex' not in cfg and isinstance(cfg.get('vfs'), dict):
    vfs = cfg['vfs']      # 只有本模块旧版写的配置才走这条路
```

空 VFS 是合法状态，不该被旧快照覆盖。同时 `push()` 不再写 `vfs`
（web 的 `config-sync.js` 从未读过它，写它只会制造第二个真相源）。

**新增 4 项测试**，其中关键的两条：

```
✓ 不混入 vfs 里的陈旧条目
✓ 空 fileIndex 不被 vfs 覆盖
```

### 数据侧

线上 `vfs` 残留已清除（59 个旧条目）。

---

## 二、P0-2：exec 开关可被页面自行打开

### 现象

`Bridge.set_exec_enabled` 是**不带下划线前缀**的 public 方法。
pywebview 会把这类方法全部暴露成 `window.pywebview.api.xxx`。

于是页面里任意一段 JS 都能：

```javascript
pywebview.api.set_exec_enabled(true).then(() => gdpy.exec('...'))
```

黑名单还在，但**门本身是敞开的** —— 默认关闭形同虚设。

### 修法

开关只由 Python 侧决定：

- 构造参数 `Bridge(exec_enabled=...)`，由 `webview_main` 从启动参数 `--enable-exec` 读取
- `set_exec_enabled` → 改名 `_set_exec_enabled`（下划线开头，pywebview 不暴露）
- 保留只读的 `exec_enabled()` 供页面查询

```
.exe --enable-exec     ← 唯一开启方式
```

插件的 exec 走另一条通道（GrantStore 授权），不受影响。

**新增 7 项测试**，含一条反向验证：

```
✓ Bridge 不暴露 set_exec_enabled
✓ _set_exec_enabled 是内部方法（JS 看不到）
✓ 开启后黑名单仍拦截
```

---

## 三、P1：容量记账虚高 19.3 MB

上一轮我用脚本删孤儿，只删了 blob 和 VFS 条目，**没走 `subtractFromRepoUsage`**，
记账仍算着已删文件的容量：

| 仓库 | 记账 | 实际 | 偏差 |
|---|---|---|---|
| c4aa | 44.6 MB | 32.0 MB | +12.6 |
| ft3j | 6.2 MB | 0 | +6.2 |
| hu15 | 0.5 MB | 0 | +0.5 |

已按当前 VFS 重算。现在**记账与实际偏差为 0**：

```
记账 109.1 MB   实际 109.1 MB   孤儿 0
```

---

## 四、P2：workflow 版本号写入取空值

```yaml
- name: 写入版本号
  run: |
    V="${{ github.event.inputs.version }}"      # ← 手动触发留空时这里是空字符串
    printf '%s\n' "__version__ = '$V'" > gdrive/_version.py
```

而「解析版本号」那一步有 fallback 到 `VERSION` 文件，**这一步没有**。
所以不带参数触发构建会写入 `__version__ = ''`。

**目前被回退链兜住**（`gdrive/__init__.py` 的 `_FALLBACK`），不会显示错误版本号，
且 `test_core.py::TestVersionConsistency` 盯着 `_FALLBACK` 与 `VERSION` 一致。

但这仍是一处隐患：回退链依赖人工同步的硬编码值。建议该步改用
`steps.ver.outputs.version`。

---

## 五、两个已知限制（本轮不修，记录）

### ① ghproxy 削弱了域名白名单

`ALLOWED_HOSTS` 里有 `ghproxy.com` / `ghproxy.net`。这两个服务的设计
就是**代理任意 URL**，于是：

```
https://ghproxy.net/https://任意公网地址
```

会通过白名单校验。目前影响有限（ghproxy 只能访问公网，碰不到内网、
也碰不到 `169.254.169.254` 元数据），但**白名单的实际强度低于其名字所暗示的**。

### ② shim 只覆盖了 fetch

shim 改写的是 `window.fetch`。页面仍可用 `XMLHttpRequest`、
`new Image().src`、navigator.sendBeacon 发起请求 —— 这些**不受白名单约束**。

所以白名单防的是"通过应用自身的网络层外传"，防不住页面直接用浏览器原生能力。
纵深防御上，真正可靠的是 CSP（Content-Security-Policy），但目前本地服务没有设置。

---

## 六、检查通过的部分

| 项 | 结果 |
|---|---|
| web/ 与线上 github_drive 一致 | ✓ `d7b3ee4d` 匹配 |
| 孤儿分片清理（覆盖上传后） | ✓ 在线上，顺序正确（新文件成功才清旧的） |
| `subtractFromRepoUsage` 有被调用 | ✓ deleteFile/deleteFolder/孤儿清理/失败清理 |
| `minChunkSize` 契约 | ✓ 线上 10MB，Python 侧不再写本地默认值 |
| release 无草稿 | ✓ v0.0.4 ~ v0.0.11 全部 `draft=false` |
| 文件名带版本号 | ✓ `Desktop_Github-Drive-Windows-V0.0.11.exe` |
| VERSION / tag / `_FALLBACK` 一致 | ✓ 0.0.11（本轮升到 0.0.12） |
| 打包资源完整 | ✓ `web/` `plugins/` 均已 `--add-data` |
| shim 内嵌 | ✓ 不依赖外部 .js 文件 |
| CI 不联网拉前端 | ✓ 只用仓库内的 web/（供应链风险可控） |
| 版本号写入在 PyInstaller 之前 | ✓ 行 122 → 行 129 |

---

## 七、测试

```
test_arch          29 ✓
test_config_sync   35 ✓  (+6)
test_core          53 ✓
test_maintain      24 ✓
test_plugins      135 ✓
test_recover       12 ✓
test_security      57 ✓
test_url           16 ✓
test_webview       67 ✓  (+7)
────────────────────────
合计              428 ✓
```

---

## 八、给 fd 检查法本身的反馈（第二轮）

上一轮我提的改进（收尾检查给查法、增加线上数据核验遍、防复发断言）
这轮都用上了，并且**确实靠它们抓到了东西**：

- 「线上数据核验」这一遍 → 抓到 P0-1（`vfs` 59 vs `fileIndex` 20）
- 「收尾检查」的打包项 → 验证了版本号写入顺序、add-data 覆盖

但也有两处仍然不足：

**① 缺「跨端字段读取方」核对。**
P0-1 的本质是"写了两个键、只有一个读取方"。我上一轮修 `config_sync` 时
专门处理了 `fileIndex`/`vfs` 字段名，却没追问"vfs 到底有没有人读"。
**修完字段兼容之后，应该顺手确认新字段的所有读取方。**

**② 缺「暴露面」检查。**
P0-2 不是代码逻辑错，是"这个方法被谁调用"的问题。
框架（pywebview / js_api / 各类装饰器）会隐式扩大方法的可见范围，
读代码时很容易忽略。以后凡是"安全开关"，都要确认它在框架层是否真的不可达。
