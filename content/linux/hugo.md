---
title: "hugo 博客使用"
date: 2021-04-21T14:46:18+08:00
draft: false
tags: ["linux","sre","ops","hugo"]
categories: ["linux"]

weight: 100
contentCopyright: MIT
autoCollapseToc: false

---

# hugo 博客使用

* 导航添加栏目，如何改变顺序
    * 是根据一个 weight 控制 about,tags,archives 的顺序，在 `config.toml` 和 `content/about.md` 里边都可以配置， `about.md` 设为100 就跑了最后去了

## 环境部署

**主题安装**

首次下载主题

`git submodule https://github.com/olOwOlo/hugo-theme-even themes/even` 添加模块主题

或者 `git clone https://github.com/olOwOlo/hugo-theme-even themes/even` 添加到仓库中

在新环境中下载主题

`git submodule update --init --recursive`
