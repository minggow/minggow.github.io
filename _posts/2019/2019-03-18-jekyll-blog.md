---
layout: post
title: Jeskll 博客
category: other
tags: [other]
excerpt:  新手搭建jeskll博客
---

## jekyll定义
jekyll是一个简单的免费的Blog生成工具，类似WordPress。但是和WordPress又有很大的不同，原因是jekyll只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如Disqus。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。

## jekyll安装
### 安装ruby
```
brew install ruby
```
可能会有报错，可参考网上解决即可

### 更新gem
其中最新的gems地址是 https://gems.ruby-china.com
```
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.com
# 确保只有 gems.ruby-china.com
```
### 安装jekyll
过程可能比较曲折，可参考 [jekyll安装的斗智斗勇](https://www.cnblogs.com/hannahgu/p/5368726.html) 。最后安装成果后
```
$ sudo gem install jekyll
.....
$
Done installing documentation for http_parser.rb, eventmachine, em-websocket, concurrent-ruby, i18n, rb-fsevent, ffi, rb-inotify, sass-listen, sass, jekyll-sass-converter, ruby_dep, listen, jekyll-watch, kramdown, liquid, mercenary, forwardable-extended, pathutil, rouge, safe_yaml, jekyll after 58 seconds
22 gems installed
```
完成jekyll安装。
搭建一个博客：
```
$ jekyll new myblog
$ cd myblog
$ jekyll serve
```
如果有问题可参照 [jekyll 完整安装教程](https://blog.csdn.net/joelcat/article/details/78642434)，最后大功告成。

## Yummy-Jekyll主题
### 定义
> A Simple, Bootstrap Based Theme. Especially for developers who like to show their projects on website and like to take notes. There are also some magical features to discover.

有道翻译
> 一个简单的，基于引导的主题。特别是对于那些喜欢在网站上展示他们的项目并且喜欢做笔记的开发者。还有一些神奇的功能有待发现。

### 安装
参照 [Yummy-Jekyll安装](https://github.com/DONGChuan/Yummy-Jekyll#install-and-setup)，
首先安装bower
```
$ brew install bower
```
依赖icu4c和node，整个过程比较久。
>  - Run bower install to install all dependencies in bower.json
> - Run bundle install to install all dependencies in Gemfile
> - Update _config.yml with your own settings.
> - Add posts in /_posts
> - Commit to your own Username.github.io repository.

然后可以参照一些优秀博客的样例fork，然后修改整理自己的博客。
[微笑哥的博客](http://www.ityouknow.com/other/2016/06/10/Blog-With-Jekyll.html)
[Yummy-Jekyll](https://github.com/DONGChuan/Yummy-Jekyll)

## 使用总结
安装过程比较曲折，各种的问题出现。如果仅仅需要搭建一个博客模板，可以只fork优秀的模板即可，不需要安装。

学习掌握markdown，然后就可以开始自己的总结沉淀之旅，任何伟大的事情都需要一个不起眼的开端，努力加油。

虽然比较费时间，用了将近一天的时间解决各种问题。个人不是特别爱较真，工作和学习中的问题总是似是而非，学习和总结内容并不深刻。这是一个开端，希望自己能够努力坚持下去，爱较真才真正的态度。


参考
- [RubyGems - Ruby China](https://gems.ruby-china.com/)
- [jekyll 完整安装教程](https://blog.csdn.net/joelcat/article/details/78642434)
- [jekyll安装的斗智斗勇](https://www.cnblogs.com/hannahgu/p/5368726.html)
- [微笑哥的博客](http://www.ityouknow.com/other/2016/06/10/Blog-With-Jekyll.html)
- [Yummy-Jekyll](https://github.com/DONGChuan/Yummy-Jekyll)

