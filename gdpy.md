---
title: gdpy
---

# gdpy · GitHub Drive 桌面版

Python + Tkinter 写的桌面客户端。**这是新生态，不是网页版的移植。**

- 在线主页：<https://cool-zimo.github.io/gdpy/>
- 源码：<https://github.com/Cool-zimo/gdpy>
- 可执行文件：**不入库**，由 GitHub Actions 编译后放 Release

---

## 0. 为什么做桌面版

网页版跑在浏览器沙箱里，有两件事**永远做不到**：

1. 碰不到文件系统（不能同步到本地目录）
2. 起不了进程（不能下载完自动调用本地工具）

「把网盘文件同步到本地」「下载完自动转码」这类需求，网页版无论怎么写都实现不了。
桌面版补的就是这块。

所以插件系统**刻意不与网页版兼容**：网页插件是 HTML+JS 跑在沙箱里，
桌面插件是 Python 能直接 `open()` 和 `subprocess.run()`。兼容了反而失去意义。

---

## 1. 整体架构

```
main.py                  启动入口（Tkinter 缺失时给明确提示）
gdrive/
  core/
    api.py        GitHub REST 封装 —— 只用标准库 urllib
    config.py     本地配置 + 容量记账 + 多账号
    vfs.py        虚拟文件系统
    transfer.py   上传/下载/分片 + 失败回滚
    share.py      分享（与网页版双向兼容）
    plugins.py    插件：声明 → 授权 → 运行时校验
  ui/
    app.py        Tkinter 主界面
    plugins_ui.py 插件管理窗口
tests/test_core.py  59 项纯逻辑测试（不需要 GUI）
```

### 模块依赖

```
ui/app.py
   ├── core/config      （配置、记账）
   ├── core/api         （网络）
   ├── core/vfs         （目录结构）
   ├── core/transfer    （上传下载）
   └── core/share       （分享）
           └── core/api

core/plugins   ← 独立，不依赖上面任何一个
```

`plugins.py` 刻意做成独立的：它只依赖 `config.app_dir()`，
不与 Drive 的任何业务逻辑耦合，将来可以单独抽出去。

---

## 2. 运行时依赖：只有标准库

`requirements.txt` 里**运行时零第三方依赖**，只有构建才装 PyInstaller。

| 需求 | 用法 |
|---|---|
| HTTP | `urllib.request`（不用 requests） |
| JSON | `json` |
| GUI | `tkinter`（系统自带） |
| 并发 | `threading` + `queue` |

**为什么不用 requests**：PyInstaller 打包时能少拖一堆依赖，
三平台构建更稳、产物更小。代价是代码稍微啰嗦一点，值得。

---

## 3. 核心模块手册

### 3.1 api.py —— 两层 API

**Contents API**（单文件读写）：简单，但每个文件一次 commit。

**Git 底层 API**（批量）：

```
createBlob → createTree(base_tree) → createCommit → updateRef
```

N 个文件**只产生 1 次 commit**。省配额、避冲突，中途失败不会留半成品。

⚠️ **代价**：必须先拿 `base_tree`，两个并发调用会互相覆盖。
这是 **commit 层面的冲突**，与文件路径是否相同无关 ——
同一仓库的写入必须串行。

`batch_upload()` 就是走这条路，分享创建时用它一次性提交
「所有文件 + index.html + share.json + README.md」。

### 3.2 vfs.py —— 虚拟文件系统

**结构是纯虚拟的**：GitHub 没有目录概念，文件夹只是 VFS 里的一行记录。

```python
files:   { "/drive_home/a/b.txt": {name,type,size,chunks,createdAt,updatedAt} }
folders: { "/drive_home/a":       {name,type,createdAt} }
```

**字段名与网页版 `js/storage.js` 完全一致** —— 这是跨端兼容的基础。

三个必须知道的细节：

**① 防御式规范化不能省**

```python
if not isinstance(vfs.get('files'), dict): vfs['files'] = {}
```

VFS 会从别的设备/网页版同步过来，可能是空对象或缺字段。
直接 `.items()` 会 KeyError 崩掉。这三行是跨端同步的必备保险。

**② `children()` 的 prefix 必须是 `base + '/'`**

曾经写成对 `DRIVE_HOME` 特殊处理成 `'/'`，结果 `rest` 变成
`'drive_home/a'`（含 `/`）→ 所有子项都被跳过 → **列表永远为空**。
测试里表现为 `children('/drive_home') == []`，很隐蔽。

**③ 删文件夹只删 VFS 一行，不碰 GitHub 文件**

真正的分片删除由调用方决定。回收站就是靠这个实现「秒删」（只改配置）。

### 3.3 transfer.py —— 分片与回滚

**分片命名必须与网页版一致**，否则桌面版读不到网页版传的文件：

```
chunk_path = "{base36(毫秒时间戳)}/{文件名}"
多分片时文件名 = "{原名}.{序号}"   （序号从 1 开始）
```

>1MB 走 Git blob API，≤1MB 走 Contents API（与网页版同一阈值）。

**失败回滚**：中途挂了要把已上传的分片删掉，否则仓库里堆垃圾 ——
占容量且无法自动回收。`_rollback()` 尽力清理，**失败只警告不抛出**，
不能因为清理失败掩盖原始异常。

**容量记账**：用本地 `config.repoUsage`，不实时查 GitHub。
查一次要一次 API，每次上传前都查太贵。

`sub_usage()` 里用了 `max(0, ...)`：记账错乱会让「仓库已满」的判断失真。

### 3.4 share.py —— 双向兼容契约

| 项目 | 约定 |
|---|---|
| 仓库名 | `gd-share-{6位hex}`（旧名 `share-*` 也要认） |
| 忽略文件 | `index.html` / `README.md` / `status.js` / `share.json` |
| 元信息 | `share.json = {files:[{name,size}], createdAt, ...}` |
| Pages | `https://{owner}.github.io/{repo}/` |

**读取分享不需要 Token** —— 分享仓库是公开的，走 `raw.githubusercontent.com`。

`list_my_shares()` 的真实来源是**账号的仓库列表**，不是本地记录。
本地记录换设备/清缓存就没了，但仓库还在 GitHub 上。
本地只用来补充 description 等元信息。

读取时优先读 `share.json`（有大小元信息），
没有就列 tree 排除系统文件 —— 兼容早期分享。

### 3.5 plugins.py —— 三层安全模型

```
1. 声明   plugin.json 里写 permissions
2. 授权   安装时弹窗；官方仓库自动授信
3. 校验   ★ 运行时每次调用都查
```

| 权限 | 风险 |
|---|---|
| `fs:read` / `fs:list` | 低 |
| `net` | 中 |
| `fs:write` | 高 |
| `exec` | ⛔ 极高（等于把终端交出去） |

**未知权限一律按最高风险处理** —— 宁可挡住，不能放行。

> ### ⚠️ 自动授信 ≠ 取消运行时校验
>
> 官方插件跳过的只是**弹窗**，`GrantStore.check()` 依然在每次调用时拦截。
>
> 如果连运行时校验也去掉，插件 A 就能冒充插件 B 调用已授权的能力 ——
> **整个权限模型形同虚设**。
>
> 弹窗是给人看的，校验是给代码执行的，两回事。

**插件 ID 绝不能由插件自己上报**：

```python
ctx = PluginContext(pid, self.grants)   # pid 从 manager 传入
```

**命令黑名单**是最后一道防线（不是权限的替代品）：

```python
BLOCKED_PATTERNS = ('rm -rf /', 'chmod 777', 'curl|', ...)   # 子串匹配
BLOCKED_COMMANDS = ('shutdown', 'mkfs', 'dd', ...)           # ★ 只在命令位置匹配
```

分成两组是有原因的。早期全用子串匹配，结果误杀了正常命令：

```
cat /logs/shutdown_report.txt   ← 路径里有 shutdown，但不是关机命令
rm a.txt                        ← rm，但不是 -rf /
```

所以命令名只在**命令位置**（行首 / `|` `;` `&&` `||` 之后）匹配，
参数组合才用子串。测试里两组都覆盖了（拦 4 个 + 放行 4 个）。

---

## 4. 与网页版的兼容契约（改之前先看这里）

| 项目 | 约定 |
|---|---|
| VFS 结构 | `files` / `folders` 两个 dict，字段名同 `js/storage.js` |
| 分片路径 | `{base36(毫秒时间戳)}/{文件名}`，多分片 `{原名}.{序号}`（从 1 开始） |
| 分片大小 | 512 KB |
| 大文件阈值 | >1MB 走 Git blob，≤1MB 走 Contents |
| 存储仓 | `drive-storage-{YYYY-MM-DD}-{4位hex}`，私有 |
| 配置仓 | `github-drive-config`（与网页版共用） |
| 分享仓 | `gd-share-{6位hex}`，公开 + Pages |

⚠️ 改这些常量之前先想清楚：改错一个就会导致**跨端读不到文件**，
而且症状是「文件凭空消失」（其实还在 GitHub 上），很难排查。

`config.py` 里的 `LEGACY_CHUNK_SIZES` 保留了 50MB/20MB/5MB 的迁移分支 ——
这是网页版历史上用过的几个值，命中就拉回 512KB。**不要删**。

---

## 5. 构建与发布

### 版本号规范

**测试版 `0.x.x`，正式版从 `1.0.0` 起。**
产物名带版本号：`Desktop_Github-Drive-Windows-V0.0.1.exe`。

### 触发方式

```
Actions → Build Desktop → Run workflow → 填版本号（留空读 VERSION 文件）
```

三平台并行构建，最后合成一个 draft Release（我确认后手动发布）。

### ⚠️ 两个 Windows 特有的坑

**① Windows Defender 会删掉 PyInstaller 生成的 exe**

表现为构建「莫名失败」，日志里看不出所以然。
因为 exe 被实时扫描误判为木马，生成后当场删除。

解决：构建前把工作目录和临时目录加入 Defender 排除项。

```powershell
Add-MpPreference -ExclusionPath "$env:LOCALAPPDATA\Temp"
Add-MpPreference -ExclusionPath "$PWD"
```

**② CRLF 会让版本号变成 `0.0.1\r`**

Windows runner checkout 会把 LF 转 CRLF，`tr -d ' \n'` **不删 `\r`**，
拼出来的文件名非法。

```bash
V=$(cat VERSION | tr -d ' \r\n')    # ★ 必须是 \r\n
```

**③ Tkinter 依赖要 `--collect-all`**

不加的话 Windows 上 exe 能生成但运行时报找不到 Tcl/Tk。

### 为什么日志要存 artifact

GitHub 的 jobs logs API 会 302 跳到 Azure Blob，
**带 Authorization 头访问会 403**，拿不到内容。
所以在 CI 里 `tee` 到文件再传 artifact，这是唯一可靠的读日志方式。

### 可执行文件不入库

体积 13~25 MB，放仓库里会让 clone 变慢。
由 Actions 编译、放 Release。README 里也写明了这一点。

---

## 6. 设计决策

### D1 · 为什么 Tkinter 而不是 Electron / Qt

| 方案 | 否决原因 |
|---|---|
| Electron | 产物 80MB+，且需要 Node 工具链 |
| PyQt | LGPL 协议 + 商业授权问题，产物也大 |
| **Tkinter** | Python 自带，产物 13~25MB，三平台 CI 一次过 |

代价是界面朴素。对一个文件管理类工具来说可接受。

### D2 · 为什么运行时零第三方依赖

PyInstaller 打包时依赖越少越稳。曾经考虑 requests，
但 urllib 完全够用 —— 唯一代价是代码稍啰嗦。

### D3 · 为什么插件不兼容网页版

兼容了就等于把桌面插件降级成沙箱里的 JS，
那做桌面版就没有任何意义了。

### D4 · 为什么官方插件自动授信

每装一个官方插件都弹窗，用户会养成无脑点确认的习惯 ——
那弹窗就失去了意义。所以把「弹窗」留给真正需要判断的第三方插件。

但**运行时校验不取消**（原因见 §3.5 的警示框）。

---

## 7. 开发者手册

### 本地运行

```bash
pip install -r requirements.txt
python main.py
```

需要系统 Tkinter：

```bash
sudo apt-get install python3-tk     # Ubuntu/Debian
```

### 测试

```bash
python tests/test_core.py
```

59 项，纯逻辑不需要 GUI。CI 里三平台都会跑。

新增测试时注意：**输出含 `✓ ✗ ⚠️` 等符号**，
必须 `sys.stdout.reconfigure(encoding='utf-8')`，
否则在 Windows 中文控制台（GBK）下会 `UnicodeEncodeError` 直接崩。

### 已踩过的坑

| 坑 | 症状 |
|---|---|
| `children()` prefix 写错 | 文件列表永远为空 |
| 命令黑名单子串匹配 | 误杀 `cat /logs/shutdown_report.txt` |
| plugin ID 由插件上报 | 权限模型形同虚设 |
| 去掉运行时校验 | 插件可互相冒充 |
| Windows GBK 编码 | 测试输出非 ASCII 符号就崩 |
| Defender 删 exe | 构建莫名失败 |
| CRLF 版本号 | 文件名非法 |
| jobs logs API 403 | 拿不到 CI 日志 |

### Tkinter 线程规则（最重要的一条）

**所有 UI 操作必须在主线程。** 网络请求放后台线程，
结果通过 `queue` + `root.after()` 回抛到主线程更新界面。

在子线程里直接改控件 = 随机崩溃，而且不报错 —— 极难排查。

`app.py` 里的 `_bg()` / `_pump()` 就是这套机制，`_pump()` 是唯一的 UI 更新入口。
