---
layout: post
title: Hello，World！
date: 2025-01-23 23:09 +0800
---

按照文档一步一步地安装依赖，重试了好几次都没发现自己漏步骤了..
- [Installation](https://jekyllrb.com/docs/installation/)
- [Jekyll on Ubuntu](https://jekyllrb.com/docs/installation/ubuntu/)

中间发现了一个zsh的配置问题，主要是少了一个docker的插件支持，直接顺带着安装docker-engine了，后面应该还要找时间重新配置一下zsh吧

github-action:切换成ssh链接以确保push
```
git remote set-url origin git@github.com:Micihelle/Micihelle.github.io.git
ssh -T git@github.com
Hi Micihelle! You've successfully authenticated, but GitHub does not provide shell access.
```

主机文件与WSL互传（“文件系统”挂载）：上传头像
```
sudo ls /mnt/*        #查看windows系统下的“目录”
cp /mnt/d/Download/favicon/* .
```

本地部署： （github page部署：记得更新以后用F5清浏览器缓存重新加载）
```
bundle exec jekyll s
```

>using the plugin [`Jekyll-Compose`](https://github.com/jekyll/jekyll-compose) to  save time of creating files:

```
bundle exec jekyll page "My New Page"
bundle exec jekyll post "My New Post"
```

<script src="https://giscus.app/client.js"
        data-repo="micihelle/micihelle.github.io"
        data-repo-id="R_kgDONufzcA"
        data-category="Announcements"
        data-category-id="DIC_kwDONufzcM4CmRtu"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>