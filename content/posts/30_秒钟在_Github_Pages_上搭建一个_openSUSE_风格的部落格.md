---
title: "30 秒钟在 Github Pages 上搭建一个 openSUSE 风格的部落格"
date: 2016-12-31T00:00:00+08:00
draft: false
---
之前的 Ghost 管理密码丢了 :-(

昨晚花了一个晚上的时间把之前丢失的部落格文章通过 web.archives.org 找回到了 2011 年。09~10 年的部落格文章等想怀旧的时候再继续找。下一步应该研究的是怎么恢复和整合评论。
不过这个坑就不一定哪天来填啦。

考虑到我博客更新的速度和访问量，我觉得还是不要把它放在论坛服务器上了。毕竟 Ghost 是用 nodejs 的，一直跑着太占资源；搬服务器就要迁移，我这种懒人，迁完论坛就不管的，
博客要么丢文章要么丢评论。还是用 github 来做博客好了。

Github Pages 是由 Jekyll 驱动的静态博客，使用 Markdown 来写文章，用 git 管理，这几个对我来说都不是很难，毕竟我是一个 Ruby 程序猿。Github 自己的文档基本无用，因为文档跟用户之间少了点东西。
它只告诉了我 Github Page 其实就是一个静态网页生成器加 HTTP 服务器，你喂 Markdown 格式的文章给它，它在 yourname.github.io 上显示出来。这用来建立一个单页面是足够的。
但是它并没有说你想做一个博客应该怎么办。因为其实那是 Jekyll 这个静态网页生成器的事情。你需要给它写模板。

所以我参考了 Smashing Magazine 上面的 [Build A Blog With Jekyll And Github Pages](https://www.smashingmagazine.com/2014/08/build-blog-jekyll-github-pages/)，找了一个现成的 Jekyll Now 模板。自己简单改了下。风格参考了 [Grover Chou](https://groverchou.com/blog) 的部落格。我觉得他那个 openSUSE Leap 风格的主题很好看，于是就抄过来了。配色和样式是抄的我们 openSUSE 中文论坛。点我[预览](https://marguerite.github.io/fulltest/)。

要是有想要用我这套主题建博客的，建立过程很简单：

1. 去 github 上面 fork 我的 marguerite.github.io，然后点 Settings，把仓库名改成 `yourname.github.io`。
2. `git clone` 你的 fork 到本地，编辑一下 _config.yml，把我的信息都改成你的。
3. 把 _posts 下面我的文章删掉，用 Markdown 写你自己的，格式是 YYYY-MM-DD-TITLE.md。支持中文标题。

完成这三步，只需 30 秒，你就有了一个 openSUSE 中文论坛风格的个人博客啦。

下面是进阶内容。

#### 自己改 Jekyll 模板

基本的 HTML/CSS 知识假设你已经了解了，静态网页不涉及 Javascript。

假设你有 Firefox 和它的 FireBug 插件。就是在网页上右键能查看 HTML 元素和它们的 CSS 的。

CSS 在 style.scss 和 "_sass" 文件夹。把网站写的漂亮是你自己的事情。W3School 上有全部 CSS 元素的资料可查。改过 wordpress 的都会的。

模板主要在 "_layouts" 和 "_includes" 文件夹里面，就是 HTML 带了一些 LIQUID 语言的变量。看了就会。

如果你想实现自己的想法的话，下面这个页面会有用：

https://jekyllrb.com/docs/variables/

#### 本地开发 Jekyll（调试）

你 99.99% 是不会遇到这种情况的。即使你自己改模板，你也可以 `git commit` 然后 `git push` 直接访问网页看效果的。
一般错误比如 CSS 出错导致的渲染不了，Github 会发邮件给你，告诉你错误在哪儿。

***只有你的 Markdown 写错了***，Github 才没法给出错误提示。这种情况下，一般自己看下文章就知道错误在哪儿了。
只有我这种一次性导入比较多，不知道究竟是哪篇文章出错了才需要本地调试的。

###### 安装 rvm

因为 openSUSE 下面的 Ruby 是系统 Ruby，普通用户直接 `gem install` 会告诉你权限错误，无法写入文件夹 
/usr/lib64/ruby/gem。还有就是为了避免把系统的 Ruby 环境搞乱套，毕竟 YaST 也用 Ruby 的。

    gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
    \curl -sSL https://get.rvm.io | bash
    
运行这两条命令就把 rvm 装好了，下面是 https://rvm.io/integration 里说的那样，把 Konsole 的配置方案里的 `/usr/bin/bash` 变成 `/usr/bin/bash --login`。再重新打开 Konsole 就能用 rvm 的 Ruby 了。

###### 安装 Ruby 2.3

    rvm install 2.3
    
###### 安装 github-pages gem

这里有一个 bug。github-pages 依赖 nokogiri，nokogiri 使用自带的静态的 libxml2.a 和 libxslt.a 编译的时候，在 64 位系统上，后两者的
安装文件夹是 xxxx/usr/lib64，但是 nokogiri 只能认出 xxxx/usr/lib，所以会报错。所以我们要先装 nokogiri 并使用系统自带的 libxml2 和
libxslt 来编译。

    sudo zypper in libxml2-devel libxslt-devel
    gem install nokogiri
    
接着再

    gem install github-pages
    
就没什么问题了。github-pages 会自动带出来 jekyll。

###### 把你的仓库做一份拷贝

因为我们要在本地用 jekyll 去渲染你的仓库，肯定会多一些你不想传到 github 上去的中间文件的。

    cp yourname.github.io local
    cd local
    
###### 编译文章并调试
    
接下来用 jekyll 去编译文章

    jekyll build --trace
    
这时如果你的文章有什么问题的，会报错。但是错误不明显，这时候我们就需要改 gem 的代码来实现了。比如：

    Conversion error: Jekyll::Converters::Markdown encountered an error while converting '_posts/2013-3-17-openshift-搭建和-IRC-频道互通的-Gtalk-群聊.md':                                                                                                  
                    #<Class:0x000000018376c8>::AmbiguousGuess
    /home/zhou/.rvm/gems/ruby-2.3.3/gems/rouge-1.11.1/lib/rouge/lexer.rb:153:in `guess': Ambiguous guess: can't decide between ["html", "shell"]

光这么看，你只能知道是哪篇文章出的错，但你不知道是在哪里。但是下面已经告诉你是 lexer.rb 这个文件出错的，我们可以改动它来多输出一些信息。

ruby 下面输出东西非常简单，`p 某个变量` 即可。这就是原理。

    cd /home/zhou/.rvm/gems/ruby-2.3.3/gems/rouge-1.11.1/lib/rouge/
    vi lexer.rb
    
lexer.rb 对应的行是这样的：

    def guess(info={})
        lexers = guesses(info)
        
        return xxx if xxx
        return yyy if yyy
        
        raise AmbiguousGuess.new(lexers)
    end
    
这是在定义一个 Ruby 函数，因为它出错了，所以我们不需要看正常情况的 `return`，只需要看不正常情况的 `raise`。

看到 AmbiguousGuess 是一个函数，它接受的输入是 lexers，可以猜到它是接受 lexers 返回对应的错误，重点还是在 lexers。

而 lexers 的定义在上面，`lexers = guesses(info)`。看到 guesses 是另一个函数，接受 info，处理过返回 lexers。那我们就猜说 info 可能就是文本。

于是：

    def guess(info={})
        p info
        lexers = guesses(info)
        
保存，再运行一遍 `jekyll build --trace`，这回的出错信息更具体了。直接把出错的内容都输出了。对应改就好。

另外如果你是想在本地改 CSS 而不是去找编译错误的话，用 `jekyll serve` 然后访问 127.0.0.1:4000 即可。



