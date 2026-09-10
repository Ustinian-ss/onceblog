# onceblog

基于 Hexo 8 + 深度定制 Landscape 主题的个人静态博客，部署在 Vercel。

- 线上地址：https://onceblog.vercel.app
- 主题：`themes/landscape`（就地修改版，升级主题会覆盖本地改动——改样式请直接改这个目录）
- 生成内容：RSS（`/atom.xml`）、站点地图（`/sitemap.xml`）、本地搜索（`/search.xml`）

## 写作

```bash
pnpm install
pnpm server        # 本地预览 http://localhost:4000
hexo new "文章标题"  # 在 source/_posts/ 生成草稿
```

文章 front-matter 至少包含 `title` 与 `date`；建议补 `tags`/`categories`（当前两篇文章均缺失，标签页为空）。

**首页摘要**：在正文中插入 `<!--more-->`，其上方内容作为首页摘要展示，否则整篇上首页。

## 构建与部署

```bash
pnpm build         # 生成 public/
```

推送到 main 后 Vercel 自动构建（`vercel.json` → `pnpm run vercel-build`）。

## 维护备忘

- 评论系统：主题内置 Valine（`themes/landscape/_config.yml` `valine.enable`），当前关闭；开启需配置 LeanCloud appId/appKey，挂载点已就绪（文章页 `.vcomment`）。
- 星空背景/搜索/暗色模式均为原生 JS，位于 `themes/landscape/layout/_partial/after-footer.ejs`。
- jQuery 仅用于移动端菜单、分享弹窗与图片灯箱（`themes/landscape/source/js/script.js`）。
