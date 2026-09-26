# 架构收口报告 · Python 退成纯桥

**日期**：2026-09-26
**版本**：gdpy v0.0.11
**结果**：删除 1417 行，新增契约层与架构守护，测试 413 项全绿

---

## ★ 先修正一个说法

我上一轮说"Python 退成纯桥还没做，现在该动这个手术"——
**这个描述不准确。**

核查后发现：**webview 模式下 Python 早就是纯桥了。**

```
webview_main.py  →  import Bridge + server
bridge.py        →  只用 core.api._ascii_safe
server.py        →  静态文件服务
```

业务模块（vfs / transfer / storage / share / config）
**只有 `ui/app.py` 在用**——也就是早已废弃的 Tkinter 旧版。

所以真正的状况不是"有两套实现在同时跑"，
而是**有一套没人用的实现还躺在那里**，我还在为它写测试、为它修 bug。

（比如 v0.0.9 修的 `minChunkSize 512KB → 10MB`，
修的是一个**没有调用方**的常量。）

这个区别很重要：
- 如果是"两套在跑" → 紧急事故，立刻停一套
- 如果是"一套是死的" → 清理，但要小心别删到活代码

**实际是后者。** 而我在删的过程中真的误删了活代码测试，见下文。

---

## 做了什么

| 动作 | 内容 | 行数 |
|---|---|---|
| 删除 | `core/vfs.py` `core/transfer.py` `core/storage.py` `core/share.py` | −1098 |
| 删除 | `ui/app.py` `ui/plugins_ui.py`（Tkinter 旧版） | −599 |
| 新增 | `core/contract.py`（契约层） | +119 |
| 新增 | `tests/test_arch.py`（架构守护） | +185 |
| 瘦身 | `config.py` 删掉 `DEFAULT_STORAGE_CONFIG` 全套 | −60 |

---

## 新架构

```
业务逻辑  →  只存在于 web/js/（前端实现，唯一真实来源）

Python    →  只做三件事：
              1. http_request   GitHub API 代理（浏览器有 CORS）
              2. 本地文件读写 / 文件对话框
              3. exec           插件能力

Python 不再实现：上传、分片、配额计算、仓库选择、分享
```

### 关键设计：不定义业务默认值

`config.py` 里整套 `DEFAULT_STORAGE_CONFIG` 删掉了。

```python
# 曾经的 bug：
#   Python 写 minChunkSize = 512KB
#   线上真实值 = 10MB（用户在网页版保存设置时写进 config.json 的）
#   → 桌面版一写回 config.json，就把用户的分片策略改掉了
```

新规则写在文件里：

> 需要契约值 → 从线上 config.json 读
> 读不到 → 报错，**不用本地默认值兜底**
>
> ★ 想加 `DEFAULT_` 常量前先问：
>   我在**定义**它，还是在**读取**它？
>   如果是定义 —— 那就是在制造下一处漂移。

---

## 契约层 contract.py

只放与 js 有**格式约定**的东西：

| 内容 | 为什么必须留 |
|---|---|
| `DRIVE_HOME` | VFS 根路径，跨端一致 |
| `_b36(n)` | 与 JS `toString(36)` 同形 |
| `is_chunk_dir(name)` | 识别 js 生成的随机分片目录 |

**`_b36` 的分量**：js 用它生成分片目录 `mt` + `_b36(Date.now())`。
56MB 文件丢失事件的根因就是这种目录不断累积、旧目录从不删除。
Python 用它**识别**这类目录（扫描孤儿），不用于生成。

### 用真 node 验证过

| n | JS `toString(36)` | Python `_b36` |
|---|---|---|
| 0 | `0` | `0` |
| 35 | `z` | `z` |
| 36 | `10` | `10` |
| 123456789 | `21i3v9` | `21i3v9` |
| 1760000000000 | `mgj6k3cw` | `mgj6k3cw` |

**旧断言是 `_b36(x) == _b36(x)`——恒真，等于没测。**
现在用 node 跑出来的真实对照值写死在测试里。

---

## 架构守护测试（29 项）

> **删代码是一次性的，退化是持续的压力。**

下一个写功能的人（包括未来的我）最自然的想法就是
"这个计算 Python 做起来方便"，然后在 `core/` 下加一个模块。
半年后又是两套实现。

这个文件是那道闸——**不是禁止写 Python，而是让它无法悄悄发生**。

```
★ core/ 下不存在业务实现模块（vfs/transfer/storage/share）
★ ui/app.py（Tkinter 旧版）已移除
★ config.py 无 DEFAULT_ 常量（用 AST 扫，不是正则）
★ config.py 不含 minChunkSize / maxRepoSize 等业务字段
★ contract.py 不碰 IO（urllib / requests / open）
★ 运行时入口不 import 业务模块
★ bridge 不含业务方法（upload / pick_repo / split_file …）
★ bridge 提供 http_request / read_file_b64 / write_file_b64 / exec_command
★ 插件只用本地能力（8 个接口白名单，不含任何 GitHub 业务能力）
```

`config.py` 那条用 **AST 扫赋值语句**，不是正则匹配——
正则会被注释里的 `DEFAULT_` 字样骗过去。

---

## ★★ 过程中自己犯的三个错

这三个都写进代码注释了，因为**都是"看起来对、实际错"的类型**。

### 错误 1：删测试连带删掉了变量定义

删 `【1】-【9】` 那段测试时，把里面定义的 `tmpdir` 一起删了，
后面的用例 `NameError`。

> **删测试不能只删"看起来无关"的段落，删完要跑一遍。**

### 错误 2：以为活代码随模块一起下线了

以为 `breadcrumb_segments` / `shorten` 随 `vfs.py` 一起没了，删了对应测试。

实际它们**早被抽到 `core/textutil.py`**，是活代码
（`tools/maintain_cli.py` 在用 `human_size`）。

> **删测试前必须确认被删的「函数」还活着，不能只看「模块」在不在。**

### 错误 3：刚消除双实现，又造了一处

我在 `contract.py` 和 `textutil.py` 各写了一份 `human_size`——
**新的双实现**，而且是在做"消除双实现"的提交里造出来的。

已合并到 `textutil.py`。判断标准写进注释了：

> 这个函数与 js 有格式约定吗？没有 → 不是契约，不放 contract.py。

---

## 验证

| 项 | 结果 |
|---|---|
| 测试 | **413 项全绿**（core 53 / arch 29 / config_sync 29 / plugins 135 / security 57 / url 16 / webview 60 / maintain 22 / recover 12） |
| 模块导入冒烟 | 10/10 保留模块全部可导入 |
| 打包资源检查 | 通过（无未声明的运行时资源） |
| 残留引用扫描 | 仅剩注释提及，无代码引用 |

收口前 417 项 → 现在 413 项。
净减 4 项：删掉的死代码测试 ≈ 新增的架构守护测试。

---

## 未做 / 待确认

| 项 | 说明 |
|---|---|
| `maintain_cli.py` 仍依赖 `Config` 类 | 它是离线维护工具，保留合理；但它读的本地 config 缓存与线上 config.json 的关系尚未厘清 |
| js 侧 `subtractFromRepoUsage` 正确性未验证 | test_security【6】标了"待验证"——记账扣减是这次 42% 偏差的另一半成因 |
| `pendingOrphanChunks` 无持久化 | 清理失败的分片只记在内存，刷新即丢 |
| 44 个恢复文件待用户确认 | 在 `/drive_home/_recovered` |
