---
layout: post
title: "在windows下通过cygwin使用octopress"
date: 2013-06-01 16:50
comments: true
categories: octopress cygwin
---

在windows下使用git和octopress还真是个麻烦事，本文记录了在windows下通过cygwin安装git和使用octopress的过程，希望对大家有所帮助。

## 准备工作
1. 下载 [cygwin](http://cygwin.com/install.html)
2. 下载 [ruby 1.9.3](http://rubyinstaller.org/downloads/)
   注：需要使用RubyInstall的方式进行安装，可在该下载页面可找到1.9.3的版本，Octopress依赖ruby1.9.3版本，其他版本可能会出现兼容性问题。
3. 下载 [DevKit](http://rubyinstaller.org/downloads/)

## 安装cygwin
安装cygwin时，默认是不安装git的，因此需要在安装时选中git，如下图所示：
![cygwin中安装git](/assets/blog/20130603/cygwin_git.jpg)
另，在使用github时，需要提供ssh key，所以在安装时也需选中openssh包。

## 安装ruby和DevKit
安装ruby时，记得选中“Add Ruby executables to your PATH”，将ruby的执行路径添加到环境变量中，安装完后接着安装DevKit。  
注意，安装DevKit这一步是必需的，因为若不安装的话，后面安装其他依赖时会出错，首先解压下载下来的DevKit包，在解压的文件夹下打开命令行执行以下命令即可完成安装。
```
ruby dk.rb init
ruby dk.rb install
```

## 获取octopress并安装相关依赖
首先从github上获取octopress源代码，不知道如何使用github的可以参考 [Windows 下使用Git管理Github项目](http://blog.fwhyy.com/posts/1709) 这篇文章。
若没有自己的分支可clone主干版本。
```
$ git clone https://github.com/imathis/octopress.git octopress
cd octopress
```

在cygwin中使用gem等命令时，会出现找不到路径的错误，此时可以使用gem.bat命令，但每次写.bat着实是件麻烦事，可通过在.bash_profile中设置别名来解决这个问题。
```
alias gem=gem.bat
alias irb=irb.bat
alias rake=rake.bat
```

安装其他依赖
```
gem install fast-stemmer
gem install bundler
bundle install
```
fast-stemmer是执行bundle install时所需的一个依赖项，若不安装的话则bundle install命令执行到大半时会失败，这个时候需要根据提示安装其他依赖项，再重复这个步骤，若出错再重复，在天朝这么坑爹的网络环境下，有时又会出现无法下载的情况，真是好不折腾。若使用goagent，在无法下载时，可设置shell通过代理上网，命令如下。
```
export http_proxy=127.0.0.1:8087
```

## 使用octopress
安装octopress主题
```
rake install
```
在本地进行预览
```
rake preview
```
在windows下使用octopress，会有一些中文编码方面的问题，还需要做一些设置。  
在.bashrc中添加下面的键值对
```
LANG=zh_CN.UTF-8 
LC_ALL=zh_CN.UTF-8
```
在ruby安装路径下找到lib\ruby\gems\1.9.1\gems\jekyll-0.12.0\lib\jekyll\convertible.rb这个文件，查找File.read，将该行代码修改为
``` ruby
self.content = File.read(File.join(base, name), :encoding => 'utf-8')
```
还有其他2个rb文件需要采用同样的方式进行修改，当在命令执行出错时可查看trace，并找到相关的文件进行修改。

关于如何使用octopress在github上搭建blog，你可以查看 [使用Octopress + Github管理blog](http://ishalou.com/blog/2012/10/15/how-to-use-octopress/) 这篇文章。

## 总结
在windows下使用为linux而生的git和为hacker而生的octopress，确实没有在ubuntu下那么方便，不过还好这些工具都是开源的，在碰到问题的时候，可以自行修改代码来解决，这确实体现了开源的优越性。昨天也在mac下搭建了octopress的blog编写环境，由于我的mac系统版本自带的ruby是1.8.7版本，需要通过rvm来安装1.9.3版本，安装的时候也折腾了一番，对比一下，发现还是在ubuntu下使用octopress最为便捷。

