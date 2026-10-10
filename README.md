# 我的技术博客

使用 Hexo 和 [Cactus](https://github.com/probberechts/hexo-theme-cactus)，配置为白色背景。

## 本地预览

```sh
npm ci
npm run dev
```

打开 http://127.0.0.1:4000 。

## 写作与构建

```sh
npx hexo new "文章标题"
npm run build
```

文章位于 `source/_posts/`，生成的静态网页位于 `public/`。`public/` 和 `node_modules/` 不提交到 Git。

站点配置在 `_config.yml`，其中 `theme_config.colorscheme: white` 控制浅色背景。可选配色包括 `white`、`light`、`classic` 和 `dark`。

第一篇文章为 `source/_posts/robotwin-introduction.md`，目前保留标题，正文留空，可直接用 Markdown 编辑器继续完善。此前的 RoboTwin 正文保存在 `source/_drafts/robotwin-introduction-original.md`，主题排版示例保存在 `source/_drafts/hello-blog.md`，草稿默认不生成网页。

此前 RoboTwin 正文的插图与视频保存在 `source/images/robotwin/`，可在后续写作时继续使用。

## 访问统计

已接入 Umami Cloud，设置位于 `_config.yml` 的 `theme_config.umami_analytics`。发布后可在 Umami 后台查看访问和来源统计。

统计域名限定为 `frankliee.github.io`，本地预览不会计入统计。以后更换域名时，需要同步修改 `domains` 配置和 Umami 中的网站设置。

当前配置负责采集统计，页面上的公开访问次数需要另外接入数据读取与展示。

## 主题来源

`themes/cactus/` 保存官方 Cactus 主题的副本，来源提交为 `7a6074d830ac5d524582f142ac524c041077aad7`。本地增加了 Umami 统计域名配置支持，主题保留原始 MIT 许可证。

## GitHub Pages

正式站点地址为 https://frankliee.github.io 。`.github/workflows/pages.yml` 会在推送到 `main` 时自动安装依赖、构建 Hexo，并将 `public/` 中的静态网页发布到 GitHub Pages，也支持在 Actions 页面手动运行。

仓库的 **Settings → Pages → Build and deployment → Source** 应设置为 **GitHub Actions**。该项目使用 Hexo，默认的 Jekyll 分支构建无法生成博客。`zf_main` 用于本地开发，需要将改动推送到 `main` 才会更新线上网站。
