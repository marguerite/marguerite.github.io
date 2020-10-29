---
title: "Ruby on Rails 学习笔记－安装和 Hello World"
date: 2012-11-11T00:00:00+08:00
draft: false
---
因为想给 Open Build Service 加 i18n 支持，所以要学习下 Ruby on Rails。

#### 什么是 Ruby on Rails？

推特上有 Ruby 程序员说是「肉夹馍」的关系，意思是其实硬菜在 Ruby。

对新手这么要求可能太苛刻了，初心者应该是「荷叶包饭」的关系，意思是你真正一开始吃的部分是 Rails。

但是要想学好 Ruby on Rails，其实应该是「鸡蛋灌饼」的关系，就是 Ruby & Rails，两手都要硬，毕竟所有的东西都是用 Ruby 写，要是没有系统学习过 Ruby 可能登不了大雅之堂。但 Rails 不熟悉啥都当 C++ 写那用它开发就没意思了。

科学的解释是这样的，Ruby 是屌丝松本行弘有感于白富美们漂亮的胸不大，胸大的腿不长，腿长的说话声音不好听，声音好听的长相又不是 S 级而自主研发的一个女神，同时兼顾了其他屌丝的需求（几乎所有的内置方法都可以由类在运行时重定义，眼前的加不是加，你说的 < 是什么 <，什么的）。gem 类似于芭比娃娃的衣橱（类似 perl 的 CPAN，python 的 PYPI，带平台的分发格式）。Rails 是其中一个 gem，一个网络框架。于是 Ruby on Rails 就有点像，Wordpress on PHP 的意思，然后你就是在 WordPress 上开发主题。。。

不要怪我吐槽它啊。。。less is more：()少做多。还有：Ruby 是让程序员「快乐」的一门语言。。。。您再想想？

#### 开发环境

教材：Ruby on Rails Tutorial 3rd editon（好象是新浪爱问上下载的）

系统：openSUSE 12.2,主要用到它的 zypper 命令安装软件包

终端：konsole

文本编辑器：kwrite

项目查看：dolphin

这里需要说下，Ruby on Rails 是一个网页框架，因此开发环境不可能太复杂。有些书里建议用 IDE，但是我不推荐，我写 php/javascript/css 从来没用过 IDE，有学习超重量级 IDE 如 vim/emacs 的时间你已经写出能用的程序了。（尤其是 Ruby on Rails 这种以傻瓜著名于世的语言）

但还是附一个 IDE 的比较表供有 IDE 情节的初学者们参考：

* emacs：不推荐。主要是安装包太大，而图形界面又太丑。不会配的话连树形结构都显示不出来。吐槽一下 Emacs fans，你们的 ctrl+p 是怎么运用自如的？
* vim：不推荐。树形结构也是要配置的。但是只喜欢在 konsole 里干活的可以用，vim 文件打开，按 i 是编辑模式，esc 是浏览模式，:wq 保存退出，:q! 退出。
* rubymine。据说是 Rubyer 很推崇的 IDE，30 天免费，所以我就直接没用。
* kdevelop-ruby。不太稳定，别的老牌发行版可能都没有。openSUSE 的包在 KDE:Unstable:Playground 源里，但是不能创建新工程（似乎是打包的问题），于是就变成了一个超重量级文本编辑器。。。
* eclipse。可以。但我没写过 java，eclipse 完全一抹黑。
* apatana。个人推荐。下载解压就能用。但很遗憾的是它的树形结构有点问题（不过好歹能看到结构像个 IDE 了），不能像 textmate 那样动态刷新，比如你在它的终端里 rails new first_app，旁边的树形结构是不能马上认出有这个文件夹的，要关掉重开。（那我还要个内置的终端干什么？）

#### 搭建 Ruby on Rails

有两种方法，一种是给我这种洁癖用的，一种是给图方便的。

先说图方便的用法：

    \curl -L https://get.rvm.io | bash -s stable –rails

安装 rvm，上面一套 combo 把 ruby 和 rails 都装到 /usr/local 了。但是还不能立即跑：

    sudo zypper in nodejs

洁癖方法：

    sudo zypper in rubygem-rails-3_2

直接全搞定。

区别是后者安装 gem 时要 sudo gem install 或者用 zypper 装 opensuse 的 devel:ruby:extenstions 仓库的预制软件包，后者不用。而后者在 rails new 的时候，最后有一个 bundler install 提示输入密码，输入后会卡住，因为 rails 没有安装权限。解决方法是看到那个就按 ctrl + c，然后打开 Gemfile 查看依赖，再用 zypper 装。或者 sudo rails new。

ruby 有个比较讨喜的地方就是它的 gem 不像没有 pip 的 python 那样，它们是被ruby 跟踪的，因此可以 uninstall，天然纯净无污染。运行 gem server 后在 0.0.0.0:8808 里可以看到所有本地 gem。之所以说第一种方法是脏方法是因为最后它会在 /usr/local 里留下一个没有被包管理器跟踪的 ruby。

总之第一种方法装的方便，第二种方法卸的干净。自行斟酌。

#### First Blood

建立 rails_project 文件夹。然后：

    rails new first_app
    rails server

然后浏览器打开 0.0.0.0:3000。

你现在是一个菜鸟 rubyer 了。

#### 基本拓扑

Ruby 主要是采用 view <==> controler <==> model 拓扑。control 是中央处理器。你在浏览器的请求先提交给 controller，然后它去 model 查找数据模型和数据库返回值，然后把它们插入到 view 当中得到 html，再将 html 返回给浏览器。

以上三者都在 first_app/apps 文件夹下。其它的文件夹都是 Ruby on Rails 标准文件夹，几乎所有项目都一样。
