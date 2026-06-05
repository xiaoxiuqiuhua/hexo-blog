---
title: 博客写文章与发布完整指南
date: 2026-06-05 12:00:00
tags:
  - Hexo
  - Stellar
  - 教程
  - 博客
categories:
  - 博客
---

> 这是一篇"自我指涉"的指南——它本身就是按下面这套流程写出来并发布到 blog.billnas.cn 的。把这篇收藏起来，下次忘了命令直接来查。

---

## 目录

1. [新建文章](#1-新建文章)
2. [本地预览](#2-本地预览)
3. [清理与重新生成](#3-清理与重新生成)
4. [发布到线上](#4-发布到线上)
5. [常用命令速查表](#5-常用命令速查表)
6. [写文章的小技巧](#6-写文章的小技巧)
7. [常见坑与排查](#7-常见坑与排查)

---

## 1. 新建文章

在博客根目录（`E:\Claude Code\blog`）打开终端，执行：

```bash
npx hexo new post "我的新文章"
```

> 标题含中文或空格时必须用**双引号**包起来，否则会被拆成多个参数。

执行后会在 `source/_posts/` 下生成 `我的新文章.md`，并自动填好 front matter 模板。文件长这样：

```markdown
---
title: 我的新文章
date: 2026-06-05 12:00:00
tags:
categories:
---
```

### front matter 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `title` | 是 | 文章标题，会显示在页面顶和浏览器标签 |
| `date` | 是 | 发布日期，按 `YYYY-MM-DD HH:mm:ss` 格式写 |
| `tags` | 否 | 标签列表，扁平多对一篇文章 |
| `categories` | 否 | 分类，建议一篇一分类 |
| `description` | 否 | 文章摘要，搜索引擎和分享卡片用 |
| `cover` | 否 | 封面图 URL（Stellar 主题支持，文章顶部大图） |
| `banner` | 否 | 横幅图（Stellar 主题） |
| `sticky` | 否 | `true` 置顶（按数字大小排） |
| `updated` | 否 | 手动指定的更新时间 |

### Stellar 主题特有的小字段

- `toc: false` —— 关闭右侧目录（长文建议开着）
- `password: xxx` —— 文章加密（Stellar 自带，输入密码才能看）
- `keywords` —— SEO 关键词，逗号分隔

---

## 2. 本地预览

```bash
npx hexo server
# 或
npm run server
```

启动后访问 `http://localhost:4000` 就能看到博客。**改 `.md` 文件保存后浏览器自动刷新**，不用重启服务。

关闭服务：终端按 `Ctrl + C`。

> 提示：服务默认只监听本机，从手机/局域网其他设备访问不了。想局域网访问加 `-p 4000 -i 0.0.0.0`。

---

## 3. 清理与重新生成

什么时候需要 `clean`：

- 改了 `_config.yml` 或 `_config.stellar.yml`
- 改了主题文件（在 `node_modules/hexo-theme-stellar/` 下）
- 文章删了但归档页/标签页还残留
- 页面渲染异常，看不出来原因

```bash
npx hexo clean          # 删除 public/ 和缓存
npx hexo generate       # 重新生成 public/
# 或一条命令搞定
npx hexo clean && npx hexo generate
```

生成产物在 `public/` 目录，**这个目录就是最终推送到 COS 的内容**，提交到 git 即可，**不要手动改里面的文件**。

---

## 4. 发布到线上

写完文章、本地预览满意后，三步走：

```bash
git add .
git commit -m "post: 我的新文章"
git push origin main
```

推送后 GitHub Actions 自动触发（`.github/workflows/deploy.yml`），流程是：

1. `actions/checkout@v4` 拉取代码
2. `actions/setup-node@v4` 装 Node 20
3. `npm install` 装依赖
4. `npx hexo generate` 生成 `public/`
5. `coscmd upload -rfs --delete ./` 推到腾讯云 COS（桶 `blog-hk-1316248323`，ap-hongkong 区域）

整个过程 **1-2 分钟**。完成后刷新 `https://blog.billnas.cn` 就能看到新文章。

### 验证发布成功

1. 打开 https://github.com/xiaoxiuqiuhua/-/actions 看最新一次 workflow 是否 ✅ 绿色
2. 浏览器硬刷新（`Ctrl + F5`）blog.billnas.cn
3. 点归档或标签页确认新文章出现

### 发布失败怎么办

| 现象 | 排查 |
|------|------|
| Actions 跑失败 | 点进 workflow 看日志末尾的红色报错 |
| 报 `coscmd config` 失败 | GitHub 仓库的 `COS_SECRET_ID` / `COS_SECRET_KEY` secrets 可能过期或被删 |
| 页面 404 | 等待 1-2 分钟，CDN 有缓存；或 `hexo clean && hexo generate && git push` 强制重建 |
| COS 报 403 | 子账号没有 `PutObject` 权限，去腾讯云 CAM 控制台检查 |

### 回滚

```bash
git revert HEAD         # 撤销最近一次 commit，生成新 commit
git push origin main    # 推送后 Actions 重新部署
```

或直接 `git reset --hard HEAD~1 && git push --force`（**慎用**，会丢历史）。

---

## 5. 常用命令速查表

| 命令 | 作用 |
|------|------|
| `npx hexo new post <name>` | 新建文章（默认布局） |
| `npx hexo new page <name>` | 新建独立页面（如 about） |
| `npx hexo new draft <name>` | 新建草稿（`source/_drafts/`） |
| `npx hexo server` | 本地预览（默认 4000 端口） |
| `npx hexo clean` | 清理 public 和缓存 |
| `npx hexo generate` | 生成静态文件到 public/ |
| `npx hexo deploy` | 部署（当前项目用 git push 走 Actions，**不推荐用这个**） |
| `npm run build` | 等价于 `hexo generate` |
| `npm run server` | 等价于 `hexo server` |
| `npm run clean` | 等价于 `hexo clean` |

---

## 6. 写文章的小技巧

### 插入图片

```markdown
![alt 描述](图片 URL)
```

图片 URL 建议先用图床托管（腾讯云 COS 已有 `blog-hk` 桶，可以专门建个 `img/` 目录做图床），不要用本地相对路径——hexo 默认不会把 `source/_posts/` 下的图片复制到 public。

### 代码块

三个反引号包起来，加语言名自动高亮：

````markdown
```bash
echo "hello"
```
````

支持的常见语言：`bash` `python` `javascript` `typescript` `yaml` `json` `markdown` `sql` `diff` 等等。

### 草稿

不想让文章立刻上线但又想保存内容：

- front matter 加 `draft: true`
- 或文件名前加下划线（如 `_半成品.md`）
- 草稿放 `source/_drafts/` 而不是 `_posts/`
- `hexo server` 默认**不显示**草稿，加 `--draft` 参数才显示

### 标签 vs 分类

- **分类**是层级（一篇文章一个分类），用于侧边栏树状导航
- **标签**是扁平（一篇文章多个标签），用于相关推荐

例：

```yaml
categories:
  - 技术
tags:
  - Hexo
  - Stellar
  - 教程
```

### 引用块做摘要

文章开头用 `>` 引用块放一句话摘要，列表页和搜索引擎会优先抓这段。

---

## 7. 常见坑与排查

### 改了 `_config.stellar.yml` 但页面没变化

→ 必须 `npx hexo clean` 再 `npx hexo generate`，hexo 不会自动检测配置文件变更。

### `npx hexo new` 报 EACCES 权限错误

→ 不要用管理员权限运行 VS Code 或终端。hexo 把文件写到 `source/_posts/`，当前用户需要有写权限。

### 第一次 `git push` 提示没有 remote

```bash
git remote add origin https://github.com/xiaoxiuqiuhua/-.git
git branch -M main
git push -u origin main
```

### 中文章名 hexo new 失败

用引号包：

```bash
npx hexo new post "我的新文章"   # ✅
npx hexo new post 我的新文章     # ❌ 会被拆成多个参数
```

### 文章正文用 Nunjucks 模板语法报错

Stellar 主题底层用 Nunjucks 模板引擎，文章正文里如果出现双花括号字面字符会被当成变量插值。两种解决方法：

- 行内代码里用 HTML 实体转义双花括号，例如写成 `` `&#123;&#123;xxx&#125;&#125;` ``（浏览器渲染时显示为 `{{xxx}}`，但 Nunjucks 看不到）
- 用 raw 块标签把整段包起来，写法是 `&#123;% raw %&#125;` 开头、内容、`&#123;% endraw %&#125;` 结尾

### COS 缓存

极少数情况下，CDN 边缘节点缓存导致新文章没立即显示。处理：
1. 等 5-10 分钟
2. 浏览器硬刷新（`Ctrl + F5`）
3. 腾讯云 COS 控制台 → 静态网站 → 刷新预热 → 输入 URL 强制刷新

---

## 写在最后

- 写完文章 = `.md` 文件 + `git push`，部署全自动
- 改主题或配置 = 必须 `hexo clean` 再 `generate`
- 线上出问题 = 看 GitHub Actions 日志，不行就 `git revert` 撤回
- 备份靠 git，文章和配置都在 `xiaoxiuqiuhua/-` 仓库，不丢

如果遇到本文没覆盖到的问题，留言或提 Issue。
