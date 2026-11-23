# xue

个人博客：**https://xuehao416.github.io/**

主要记录和分享关于 **Agent**、**RSI（Recursive Self-Improvement）** 与 **Harness Engineering** 等方向的研究笔记与技术总结。

## 本地开发与发布

```bash
# 安装依赖（需要 Node.js >= 20 与 pnpm）
pnpm install

# 本地预览
pnpm dev

# 构建站点与搜索索引
pnpm build
```

新增文章直接放在 `src/content/posts/` 目录下，推送到 `main` 分支后由 GitHub Actions 自动构建并部署到 GitHub Pages。

---

Powered by [Astro](https://astro.build/) & [Fuwari](https://github.com/saicaca/fuwari).
