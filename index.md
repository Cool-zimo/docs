---
title: 项目文档
---

# 📚 Cool-zimo 项目文档

三个纯前端应用，全部托管在 GitHub Pages，数据都存在你自己的仓库里。
**没有后端、没有数据库、没有中间人。**

| 应用 | 做什么 | 在线 | 代码量 |
|---|---|---|---|
| **FaceHub** | 端到端加密聊天 / 朋友圈 / 小程序 | <https://cool-zimo.github.io/FaceHub/> | 8660 行 |
| **GitHub Drive** | 把 GitHub 当网盘 | <https://cool-zimo.github.io/github_drive/> | 9843 行 |
| **仓鼠** | GitHub 仓库管理面板 | <https://cool-zimo.github.io/cangshu/> | — |

---

## 文档导航

### 架构与设计

- **[整体架构](architecture.md)** —— 无后端约束、仓库即数据库、
  消息流设计、三层缓存、模块依赖图。**建议先读这篇。**
- **[设计决策记录](decisions.md)** —— 13 条决策，每条都写了
  **被否决的方案和理由**。想改代码前建议看。

### 模块手册

- **[FaceHub](facehub.md)** —— crypto / api / group / attach / moments /
  ai-reply / miniapp 逐模块说明
- **[GitHub Drive](github-drive.md)** —— VFS、分片、回收站、多账号隔离
- **[仓鼠](cangshu.md)** —— 配置同步、中文安全 base64

### 开发

- **[fd 检查法](fd-check.md)** —— 改完代码必须走完的固定流程：
  三遍深度检查（静态 / 契约一致性 / 失败路径）、产物环境冒烟测试、
  三要素报告、非代码收尾检查
- **[真实使用数据报告](usage-report.md)** —— 线上 config 与 4 个存储仓实测：
  42% 是孤儿数据、记账与实际双向脱节
- **[fd 检查报告 · Drive](fd-report-drive.md)** —— 网页版 + 桌面版完整 fd 结果
- **[fd 检查法 v2 改进建议](fd-check-v2-notes.md)** —— 用过四次之后，
  哪些检查项太臃肿、哪些太简略
- **[开发者手册](developer.md)** —— 本地开发、测试技巧、发布流程、
  **10 个踩过的坑**（含两个致命 bug）
- **[账号互联](bridge.md)** —— 三个应用怎么互认登录

---

## 独立组件

**[tiny-md](https://cool-zimo.github.io/tiny-md/demo.html)** —— 零依赖
Markdown + 数学公式 + 代码高亮渲染库（MIT，可独立引用）。

```html
<script src="https://cdn.jsdelivr.net/gh/Cool-zimo/tiny-md@main/tiny-md.js"></script>
<script>TinyMD.injectCSS();</script>
```

---

## 共同的设计原则

1. **零外部依赖** —— 不引框架、不引 CDN 库。见 [D9](decisions.md#d9--为什么零外部依赖)
2. **数据主权归用户** —— 全在自己的仓库里，不经第三方
3. **公钥放公开处** —— 让异步通信成为可能。见 [D3](decisions.md#d3--长期身份密钥-vs-按会话密钥)
4. **失败的代价最小化** —— 分片要小（重试只重传一片）、轮换失败不阻断移除

---

## ⚠️ 读之前要知道的三件事

1. **私钥只在 localStorage**。换设备/清缓存 → 旧消息解不开。这是纯前端 E2E 的固有代价。
2. **GitHub 无法真正删除数据**。删掉的 comment 仍在 Git 历史里。
   FaceHub 里是密文所以影响小，但要心里有数。
3. **Token 权限很大**。三个应用的 token 都能读写/删除你名下的仓库。用完建议 revoke。
