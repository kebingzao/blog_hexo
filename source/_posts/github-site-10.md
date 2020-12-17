---
title: github建站系列(10) -- 增加 algolia 的搜索功能
date: 2018-05-08 16:37:24
tags: github
categories: github建站系列
---
## 前言
随着文章写的越来越多，有时候要找一篇文章，可能要找很久，所以我们将为 blog 增加文章的搜索功能， 还是一样我们直接接入第三方服务即可，不需要自己做 server 端，能白蹭就不自己写代码。 

我们用这个服务 [Algolia](https://www.algolia.com/)， 详细的操作文档可以查看 [next 接入 Algolia 搜索功能](https://theme-next.iissnan.com/third-party-services.html#algolia-search)

文档其实写的很清楚，所以我们按照文档来即可

## 操作
前往 [Algolia](https://www.algolia.com/) 注册页面，注册一个新账户。 可以使用 GitHub 或者 Google 账户直接登录，注册后的 14 天内拥有所有功能（包括收费类别的）。之后若未续费会自动降级为免费账户，免费账户 总共有 10,000 条记录，每月有 100,000 的可以操作数。注册完成后，创建一个新的 Index，这个 Index 将在后面使用。
<!--more-->

我们用 github 第三方登录之后，创建一个 index

![1](1.png)

创建好了之后，就安装一个插件扩展
```text
npm install --save hexo-algolia
```
安装完之后，在 Algolia 服务站点上找到需要使用的一些配置的值，包括 `ApplicationID`、`Search-Only API Key`、 `Admin API Key`。

注意，Admin API Key 需要保密保存。点击 ALL API KEYS 找到新建INDEX 对应的key， 编辑权限，在弹出框中找到 ACL 选择勾选 Add records, Delete records, List indices, Delete index权限，点击update更新。

![1](2.png)

接下来编辑 站点配置文件， 新增以下配置（注意，这个就是 hexo 的 _config.yml, 而不是 next 主题的 _config.yml）
```text
algolia:
  applicationID: KGVI6VPIBI
  apiKey: 41c2ccc830ca18xxx6bf1e50a03f44
  indexName: test_blog
  chunkSize: 5000
```
当配置完成，在站点根目录下执行
```text
$ export(windows 为 set) HEXO_ALGOLIA_INDEXING_KEY=Search-Only API key
$ hexo algolia
```
来更新 Index。请注意观察命令的输出。
```text

F:\airdroid_code\github\blog_hexo>set  HEXO_ALGOLIA_INDEXING_KEY=41c2ccc830ca18xxx6bf1e50a03f44

F:\airdroid_code\github\blog_hexo>hexo algolia
INFO  [Algolia] Testing HEXO_ALGOLIA_INDEXING_KEY permissions.
INFO  Start processing
WARN  ===============================================================
WARN  ========================= ATTENTION! ==========================
WARN  ===============================================================
WARN   NexT repository is moving here: https://github.com/theme-next
WARN  ===============================================================
WARN   It's rebase to v6.0.0 and future maintenance will resume there
WARN  ===============================================================
INFO  [Algolia] Identified 8 pages and posts to index.
INFO  [Algolia] Indexing chunk 1 of 1 (50 items each)
INFO  [Algolia] Indexing done.
```
这样就创建索引成功了。 最后更改主题配置文件 (next 的 _config.yml)，找到 Algolia Search 配置部分：
```text
# Algolia Search
algolia_search:
  enable: true
  hits:
    per_page: 10
  labels:
    input_placeholder: Search for Posts
    hits_empty: "We didn't find any results for the search: ${query}"
    hits_stats: "${hits} results found in ${time} ms"
```
将 enable 改为 true 即可，根据需要你可以调整 labels 中的文本。这样就完成了

## 测试
接下来就测一下， 点击搜索就可以出现搜索面板了

![1](5.png)

这样就正常了。

## 注意事项
之前有在 deploy 到线上的时候，还会遇到一个 搜索的样式错乱的问题。后面的解决方法，就是先 `hexo clean` 清掉所有的 `public` 文件。 然后再用 `hexo deploy` 重新部署。

注意，如果后面有新的文章，那么就要重新调用 `hexo algolia` 重新刷新索引，不然新的文章索引就找不到了

---
github 建个人站点系列文章:
{% post_link github-site-1 %}
{% post_link github-site-2 %}
{% post_link github-site-3 %}
{% post_link github-site-4 %}
{% post_link github-site-5 %}
{% post_link github-site-6 %}
{% post_link github-site-7 %}
{% post_link github-site-8 %}
{% post_link github-site-9 %}
{% post_link github-site-10 %}
{% post_link github-site-11 %}
{% post_link github-site-12 %}
{% post_link github-site-13 %}
{% post_link github-site-14 %}
{% post_link github-site-15 %}
{% post_link github-site-16 %}
{% post_link github-site-17 %}
