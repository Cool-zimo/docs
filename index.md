---
title: 项目文档
---

# 📚 Cool-zimo 项目文档

三个纯前端应用，全部托管在 GitHub Pages，数据都存在你自己的仓库里。

| 应用 | 做什么 | 在线地址 |
|---|---|---|
| **FaceHub** | 端到端加密的聊天 / 朋友圈 / 小程序 | <https://cool-zimo.github.io/FaceHub/> |
| **GitHub Drive** | 把 GitHub 当网盘用 | <https://cool-zimo.github.io/github_drive/> |
| **仓鼠** | GitHub 仓库管理面板 | <https://cool-zimo.github.io/cangshu/> |

## 文档

- [FaceHub](facehub.md) —— 加密原理、仓库布局、朋友圈、小程序、AI 自动回复
- [GitHub Drive](github-drive.md) —— 分片上传、仓库结构、并发实测
- [仓鼠](cangshu.md) —— 配置同步、权限、批量管理
- [账号互联](bridge.md) —— 三个应用怎么互认登录

## 共同点

- **纯静态**，没有后端、没有埋点
- Token 只存在浏览器 `localStorage`，只和 GitHub 官方 API 通信
- 数据都在你自己的仓库里，不经过任何第三方服务器

## 独立组件

- [tiny-md](https://cool-zimo.github.io/tiny-md/demo.html) —— 零依赖 Markdown +
  数学公式 + 代码高亮渲染库（MIT，可独立引用）
