---
title: FaceHub 模块手册
---

# 💬 FaceHub · 重要模块手册

> 面向需要改动代码的人。所有描述均对照 `js/` 下实际源码。

- 仓库：`Cool-zimo/FaceHub`
- 代码量：8660 行（不含测试）

---

## 1. crypto.js（673 行）— 加密核心

### 算法选型

| 环节 | 算法 | 参数 |
|---|---|---|
| 密钥协商 | **ECDH** | 曲线 **P-256** |
| 密钥派生 | **HKDF** | SHA-256，salt 用双方公钥，info=`'aes'` |
| 消息加密 | **AES-GCM** | 256 位，IV 12 字节随机 |

> ⚠️ 早期文档里写过"X25519"，**是错的**。源码 `crypto.js` 用的是 P-256。
> 两者都是 ECDH，但曲线不同、公钥格式不同，混用会导致永远协商不出相同密钥。

### 为什么必须 HKDF

ECDH 输出的原始比特**不能直接当 AES 密钥**——长度不合规、随机性分布不够均匀。

```
ECDH 原始比特 → importKey('raw', bits, 'HKDF')
             → deriveBits(HKDF, 256)   ← extract+expand
             → importKey('raw', prk, 'AES-GCM')
```

### 长期身份密钥（关键设计）

```
登录 → ensureIdentity(login)
     → 生成 P-256 密钥对
     → publishIdentity() 写进 facehub-{login}/pk.json（公开仓库）
```

**为什么**：早期密钥是**按会话生成**的，对方必须先打开应用把公钥写进会话仓库，我才能加密——所以永远"要等双方上线"。

改成长期身份密钥后，任何人想给我发消息，直接读我公开仓库的 `pk.json` 就行。**我不需要在线。**

这就是 Signal prekey / PGP 公钥服务器的同一个思路。

### 消息格式

```
E2E1.<iv_base64>.<ciphertext_base64>
```

- `E2E1` 是版本号前缀，便于以后换算法
- 解密时若不以 `E2E1.` 开头，**原样返回并标记 `plain:true`**（兼容未加密的旧消息）
- GCM 是 AEAD：篡改会直接解密失败，不会解出乱码

**每条消息用随机 IV**，所以同一句话发两次密文完全不同——不泄露"这两条消息内容相同"。

### 密钥存储

```
localStorage
├── fh:id:{login}        身份密钥（重要，丢了全解不开）
└── fh:sk:{login}/{room} 会话密钥（旧式，兼容）
```

`exportBackup()` 导出 **v2** 格式：

```jsonc
{
  "v": 2,
  "type": "facehub-keys",
  "identities": { "cool-zimo": { "priv": "...", "pub": "..." } },
  "rooms": { ... },
  "exportedAt": "..."
}
```

⚠️ **备份是明文的**。注释里写明了理由：用口令加密会引入"忘了口令更惨"的新问题。界面必须提示"谁拿到谁就能解密"。

### 指纹诊断

```js
await Crypto.fingerprint(pubB64)  // → 'a1b2c3'
```

**为什么需要**：密钥不同步时两边算出的共享密钥不同，表现为"能加密但解密全失败"，**而且没有任何报错**。打印指纹一比对就知道是不是同一对公钥。

---

## 2. api.js（592 行）— REST 封装 + 缓存

### 三个缓存 key 必须一起清

```js
// writeFile 末尾
this._ls('fh:etag:' + owner + '/' + repo, null);        // 目录 ETag
this._ls('fh:c:' + owner + '/' + repo + '/' + path, null);        // 内容（文本）
this._ls('fh:c:' + owner + '/' + repo + '/' + path + '|b64', null); // 内容（base64）
this._ls('fh:e:' + owner + '/' + repo + '/' + path, null);        // 文件 ETag
```

**只清第一个会出 bug**（写完立刻读拿到旧值）。详见[整体架构](architecture.md#一个真实的坑)。

### readLargeFile

大文件走 blob API（`/git/blobs/{sha}`），避免 contents API 的 1MB 限制。

### 消息相关

| 方法 | 说明 |
|---|---|
| `messages(owner, repo, issueNumber, since)` | 增量拉，`since` 传时间戳 |
| `sendMessage` | 发一条 |
| `deleteMessage` | 删 comment |
| `clearMessages` | 批量清空（带进度回调） |

---

## 3. group.js（495 行）— 群聊与密钥分发

### 群密钥分发链

```
创建者生成群密钥 gk
  → 用「自己私钥 + 自己公钥」wrap 一份存 gk/{me}.json
  → 给每个成员：读对方身份公钥 → wrap → 存 gk/{member}.json
```

`wrapGroupKey(gk, myLogin, repo, peerPub)`：
用「我的私钥 + 对方公钥」ECDH 出共享密钥，加密 gk。

`unwrapGroupKey(wrapped, myLogin, repo, wrapperPub)`：
必须用**分发者的公钥**解——不能写死创建者，因为补发的可能不是创建者。

**所以每个 `gk/{login}.json` 都要带 `byPub`（分发者公钥）**：

```jsonc
{ "for": "feng-zimo", "gk": "E2E1...", "by": "cool-zimo", "byPub": "BLo5..." }
```

### 移除成员的五步顺序（重要）

```
① 把人写进成员列表      ← 必须在分发之前
② 分发群密钥给 TA
③ 轮换群密钥            ← 被移除者解不开之后的消息
④ 写回成员列表
⑤ 撤掉仓库 collaborator ← 必须放最后！
```

**第 ⑤ 步为什么放最后**：前面四步都需要写权限，先撤权限就全做不了了。

**第 ① 步为什么在最前**：如果对方没发布过公钥，分发会失败；先把人写进列表，至少状态是一致的。

**轮换失败不阻断移除**——至少他已经失去仓库访问权限了（`try/catch` 里只 warn）。

### 为什么轮换现在可行

因为**身份公钥始终可读**（公开仓库）。早期设计里读不到对方公钥就没法重新分发，轮换做不了。

---

## 4. attach.js（858 行）— 附件与分片

### 实测得出的常量

```js
COMPRESS_OVER: 200 * 1024,      // 超过就压缩图片
CHUNK:         2 * 1024 * 1024, // 单片 2MB
SPLIT_AT:      4 * 1024 * 1024, // 超过 4MB 才分片
MAX_SIZE:      200 * 1024 * 1024,
```

**这些数字是实测出来的，不是拍脑袋**：

```
1MB ✓ 7.8s   10MB ✓ 9.3s   15MB ✓ 11.3s   25MB ✓ 12.8s
40MB ✗ 422 "file is too large to be processed"
```

单次 PUT 安全线在 **30MB** 左右。

**但分片的价值不只是突破上限**：

> 10MB 一次请求要 9.3 秒，中途网络抖一下全废；
> 切成 2MB 五片，每片约 2 秒，失败只重传那一片。

所以取 2MB（base64 后 2.7MB）。

### ⚠️ 分片必须串行

```
并发 5 个不同路径 → 3 成功 / 2 失败
失败原因: 409 "is at ab1cabd... but expected ..."
```

**即使路径不同也冲突**——冲突在 commit 层面，不在文件层面。每次 commit 基于仓库当前 HEAD，并发提交互相覆盖。

**结论：写入串行，读取并发。**

### 图片压缩

原图动辄几 MB，base64 后更大（+33%）。超过 200KB 就压缩。

---

## 5. moments.js（520 行）— 朋友圈

### 关系自动派生

```js
REL_CACHE_MS: 10 * 60 * 1000,   // 缓存 10 分钟
REL_MAX:      30,               // 上限 30 人
```

**来源**：私聊对象 + 群成员。**不用手动关注**——都已经在聊天了，再点一次关注是多余的（微信也没这个动作）。

**为什么必须缓存**：不缓存的话每刷一次朋友圈都要读所有群的 `group.json`，群一多请求数就爆了。

`relations(login)` 的结果缓存 10 分钟，`addRelationChanged()` 在关系变动时手动失效。

### 评论配图的存储位置

```
comments/{postId}/{ts}-{login}.json
{ login, text, imgs:[{p,n,t,s}], replyTo, ts }
```

**图存在评论者自己的仓库 `m/` 下，不是帖子作者的仓库**。两个原因：

1. 评论别人的帖子不该需要对方仓库的写权限
2. 只存路径引用，不存 base64（否则一条带图评论是几 MB 的 JSON）

⚠️ **取图 URL 必须用评论者的 login**，不是帖子作者的。用错会 404。代码里单独有 `_commentImgUrl`。

---

## 6. ai-reply.js（367 行）— AI 自动回复

### 五重防重复

| 机制 | 防什么 |
|---|---|
| 基线时间戳 | 首次开启只记最新一条，**不会**把历史回一遍 |
| 已处理 id 持久化 | 刷新/换标签页不重来（FIFO 上限 500） |
| 并发锁 | 轮询 5s 一次，AI 要好几秒，不锁就并发打 API |
| 失败次数上限 | 网络坏了不无限重试 |
| 过滤 | 不回自己发的（大小写不敏感）、不回解不开的、不回空消息 |

### 三重防无限循环

两边都开 = A 回 B、B 回 A，死循环。**实测 3~7 秒一条**。

```js
maxChain: 3,             // 连续上限
cooldown: 20000,         // 冷却 20s
stopOnRateLimit: true    // 429 立刻停
```

**清零判据**（两个坑都踩过）：

```
✗ 看"最后一条是不是我发的"  → 循环时 target 一直是对方的，永远不成立
✗ 放在上限判断之后          → chain 满了直接 return，永远走不到清零

✓ 放在上限判断之前
✓ 判据："上次自动回复之后，我有没有发过消息"
```

---

## 7. miniapp.js（199 行）— 小程序

发现规则（三选一即可）：

1. 仓库名以 `fhapp-` 开头
2. 根目录有 `fhapp.json`

`fhapp.json` 缺失时用仓库名兜底。搜索走 GitHub 仓库搜索（**30 次/分钟**，注意比 core 限速严得多）。

---

## 8. 图标系统

`icons.js` 用 `data-ico` 占位 + `mountIcons()` 渲染内联 SVG。

**为什么**：不引图标字体（要额外加载资源），也不在 HTML 里写死 SVG（太长）。

⚠️ **新加面板后要记得调 `mountIcons()`**，否则图标是空白。
