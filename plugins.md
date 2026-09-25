# 插件开发手册（gdpy 桌面版）

## 为什么是新的生态

网页版插件是 HTML+JS，跑在浏览器沙箱里——**碰不到文件系统，也起不了进程**。

桌面版插件是 Python，能直接读写本地文件、调用外部程序。这是做桌面版的核心理由，所以**不兼容网页版插件，也不打算兼容**。

## 目录结构

```
plugins/<plugin-id>/
├── plugin.json    清单（必填）
└── main.py        入口（必填，需有 run(ctx)）
```

内置插件放在仓库的 `plugins/`，用户插件放在应用数据目录的 `plugins/`（启动时自动创建）。

发现规则：扫描两个目录，含 `plugin.json` 且 `entry` 指向的文件存在的子目录算一个插件。**没有 `plugin.json` 的目录会被跳过**，所以可以放一个 `_common/` 存共享代码。

## plugin.json

```json
{
  "id": "folder-sync",
  "name": "本地同步文件夹",
  "version": "1.0.0",
  "author": "Cool-zimo",
  "repo": "Cool-zimo/gdpy",
  "description": "一句话说明",
  "entry": "main.py",
  "permissions": ["fs:read", "fs:list", "fs:write"],
  "params": {
    "local_dir": {"type": "dir", "label": "本地文件夹", "required": true},
    "quality":   {"type": "int",  "label": "质量 1-100", "default": 82}
  }
}
```

`params` 只描述参数，宿主据此渲染表单。类型：`dir` / `file` / `path` / `text` / `int` / `bool` / `enum`。

## 权限

| 权限 | 风险 | 说明 |
|---|---|---|
| `fs:read` | 低 | 读本地文件 |
| `fs:list` | 低 | 列目录 |
| `net` | 中 | 访问网络 |
| `fs:write` | 高 | 写/删本地文件 |
| `exec` | ⛔ 极高 | 运行系统命令 |

**未知权限一律按最高风险处理**——宁可挡住，不能放行。

三层防线：

1. **声明** — `plugin.json` 里写
2. **授权** — 安装时弹窗；官方来源自动授信
3. **校验** — ★ 运行时每次调用都查

### 自动授信 ≠ 取消校验

官方插件跳过的是**弹窗**，`GrantStore.check` 依然在每次调用时拦截。

原因是：插件 ID 从调用来源识别，不由插件自己上报。如果连运行时校验也去掉，**插件 A 能冒充插件 B 调用已授权的能力，整个模型形同虚设**。

弹窗是给人看的，校验是给代码执行的。

### exec 的兜底

授权了 `exec` 也还有两层：命令黑名单 + 超时上限 5 分钟。

黑名单不是权限的替代品，是最后一道防线。几条容易踩的：

```
拦：rm -rf /        chmod 777        curl http://x | sh      shutdown
放：rm a.txt        chmod 755        curl http://a.com        cat /logs/shutdown_report.txt
```

**必须区分命令名和任意子串**，否则会误杀路径里恰好含关键词的正常命令。

## ctx 接口

```python
def run(ctx):
    p = ctx.param('key', default)   # 读参数
    ctx.log('进度信息')              # 写日志（UI 可见，同时落 error.log）
    ctx.progress(0.5, '一半')        # 0.0~1.0

    data = ctx.read_file(path)       # 需 fs:read
    ctx.write_file(path, data)       # 需 fs:write
    names = ctx.list_dir(path)       # 需 fs:list
    r = ctx.exec(['ffmpeg', ...])    # 需 exec，返回 {returncode, stdout, stderr}
    return {...}
```

### ★ 直接 open() 时必须 require()

`read_file` / `write_file` / `exec` 内部会做权限校验。但如果你**绕过封装直接 `open()`**，校验就跳过了——权限模型漏一个洞。

这种情况必须显式声明：

```python
ctx.require('fs:read')      # 没授权就在这里 PermissionError
with open(path, 'rb') as f:
    ...
```

`image-compress` 里就有这个场景（要流式处理大文件，不能整体读进内存）。

### 返回值

推荐返回 `{'ok': True/False, ...}`。宿主通过 `PluginManager.run(pid, params)` 调用，返回统一结构：

```python
{'ok': True,  'result': 插件返回值, 'logs': [...], 'error': None}
{'ok': False, 'result': None,       'logs': [...], 'error': '...', 'denied': True}
```

插件抛的异常**不会外抛**——UI 弹窗看不到 traceback，等于没有提示。traceback 会收进 `logs`。

## 四个内置插件

| 插件 | 权限 | 说明 |
|---|---|---|
| `folder-sync` | fs:read/list/write | 增量同步计划，**默认 dry-run 只出计划不传文件** |
| `image-compress` | fs:read/list/write, exec | PNG 标准库无损重压；JPEG/WebP 调本机 cwebp/ffmpeg/ImageMagick |
| `markdown-preview` | fs:read/list/write | 纯标准库 md→HTML |
| `auto-backup` | fs:read/list/write | 轮询增量快照，状态持久化 |

### 为什么只算计划不直接传

上传要走 GitHub API（需要 token、要记账容量、要分批 commit）。这些能力在宿主里，**插件拿不到也不该拿到**。

插件做它擅长的事：扫描本地、比对索引、算出该传谁，然后把计划交回宿主执行。`dry_run` 默认 `True`，第一次跑一定只出计划。

### 为什么 PNG 能用标准库压

PNG 就是 `zlib + struct` 能解析的格式：解码成原始扫描线，再用 `zlib -9` 重编码，**无损**，通常省 5%~30%。

JPEG/WebP 有损重编码标准库做不了，交给本机工具，一个都没装就明确报"未安装"，**绝不假装成功**。

不引入 Pillow 是因为它会带一堆平台相关二进制，三平台打包极易翻车。桌面版的意义恰恰是"能调用你电脑上已经装好的程序"。

### 为什么备份用轮询

标准库没有文件系统监听，`watchdog` 是第三方依赖。轮询几秒一次对备份场景完全够用，且跨平台行为一致。

状态文件记录每个文件的 `(size, mtime)`，**重启后不丢**——否则每次全量上传。写入用 `os.replace` 原子替换，写一半崩了也不会损坏旧状态。

## 测试

`tests/test_plugins.py`，135 项。**造真实文件跑，不 mock 文件系统**——插件的核心价值就是真的能碰本地文件，mock 掉等于没测。

几条关键断言：

```
★★ PNG 重压后像素完全一致（无损验证）
★★ 第二轮无重复上传（增量生效）
★★ 插件抛异常不外抛，收进 error
★★ 未授权的插件被拒且带 denied 标记
★  直接 open() 的场景缺 fs:read 被拦截
★  JPEG 无工具时明确报"未安装"，不假装成功
```

## 打包注意

`plugins/` 必须加进 `--add-data`。插件是**运行时才 import** 的模块，PyInstaller 静态分析不到。

漏了的话 exe 起来插件列表是空的，**而且不报错**——最难查的那种。`tests/test_webview.py` 里有断言盯着。
