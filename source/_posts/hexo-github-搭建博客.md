---
title: hexo github 搭建博客
date: 2022-01-19 12:31:33
tags: 
copyright: true
---

这里主要分享，基于hexo和github搭建博客的过程，避免采坑。

# hexo

源码地址: [https://github.com/hexojs/hexo](https://github.com/hexojs/hexo)

<!-- more -->

## 安装
```bash
$ npm install hexo-cli -g
```

## 常用命令
详见：[https://hexo.io/docs/commands](https://hexo.io/docs/commands)
```bash
hexo init # 初始化项目
hexo g # hexo generate 的缩写 构建静态文件
hexo s # hexo server 的缩写 启动服务
hexo help # 查看帮助
hexo version # 查看版本
hexo new [layout] <title> # 创建文章
hexo d # deploy 配合发布脚本发布项目
```

## 部分配置

`_config.yml`中部分配置
```yaml
# Site
title:                  # 博客名称
subtitle:               # 博客子标题
description:            # 作者描述
keywords:               # 站点关键词，用于搜索优化
author:                 # 博主名
language: zh-CN         # 站点语言
timezone: Asia/Shanghai # 时区

# Directory
public_dir: ./ # 如果当前项目直接提交github，可以考虑改成这样，让index.html直接在根目录
```

## 服务启动
正常来说，配置校验成功后输出如下
```bash
INFO Validating config
INFO ==================================
  ███╗   ██╗███████╗██╗  ██╗████████╗
  ████╗  ██║██╔════╝╚██╗██╔╝╚══██╔══╝
  ██╔██╗ ██║█████╗   ╚███╔╝    ██║
  ██║╚██╗██║██╔══╝   ██╔██╗    ██║
  ██║ ╚████║███████╗██╔╝ ██╗   ██║
  ╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝   ╚═╝
========================================
NexT version 8.9.0
Documentation: https://theme-next.js.org
========================================
INFO Start processing
INFO Hexo is running at http://localhost:4000/ . Press Ctrl+C to stop.
```

## 缓存清理

使用过程中，有时候发现文章回退了内容，但是生成的还是之前的内容。主要是因为`db.json`数据问题，删除此文件重新进行操作即可。

# 主题设置

本文主要使用的hexo主题为`hexo-theme-next`

## 主题安装

主题地址
- 最新: [https://github.com/next-theme/hexo-theme-next](https://github.com/next-theme/hexo-theme-next)
- 老版: [https://github.com/iissnan/hexo-theme-next](https://github.com/iissnan/hexo-theme-next)

设置主题，修改`_config.yml`
```yaml
theme: next
```

拷贝主题配置，由于我是新版，使用如下命令
```bash
cp node_modules/hexo-theme-next/_config.yml _config.next.yml
```

老版的拷贝方式如下：
```bash
cp themes/next/_config.yml _config.next.yml
```

## 切换Scheme

Scheme 是 NexT 提供的一种特性，借助于 Scheme，NexT 为你提供多种不同的外观。同时，几乎所有的配置都可以 在 Scheme 之间共用。目前 NexT 支持以下 Scheme：

-   **Muse** - 默认 Scheme，这是 NexT 最初的版本，黑白主调，大量留白
-   **Mist** - Muse 的紧凑版本，整洁有序的单栏外观
-   **Pisces** - 双栏 Scheme，小家碧玉似的清新
-   **Gemini** - 左侧网站信息及目录，块+片段结构布局  
    
修改`_config.next.yml`，我目前使用`Gemini`
```yaml
scheme: Gemini
```

## 菜单设置
修改`_config.next.yml`内容，`||`前面为对应路径，后面为设置的icon
```yml
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
  # schedule: /schedule/ || fa fa-calendar
  # sitemap: /sitemap.xml || fa fa-sitemap
  # commonweal: /404/ || fa fa-heartbeat
```

# 搜索插件

插件源码地址：[https://github.com/wzpan/hexo-generator-search](https://github.com/wzpan/hexo-generator-search)


## 安装
```bash
$ npm install hexo-generator-search --save
```

## 配置

在 `_config.yml`中加入

```yaml
search:
  path: search.xml
  field: post
  content: true
  template: ./search.xml
```

修改`_config.next.yml`
```yaml
local_search:
  enable: true
```

# seo插件

插件源码地址：
- 其它：[https://github.com/hexojs/hexo-generator-sitemap](https://github.com/hexojs/hexo-generator-sitemap)
- 百度：[https://github.com/coneycode/hexo-generator-baidu-sitemap](https://github.com/coneycode/hexo-generator-baidu-sitemap)

## 安装
```bash
$ npm install hexo-generator-feed --save
```