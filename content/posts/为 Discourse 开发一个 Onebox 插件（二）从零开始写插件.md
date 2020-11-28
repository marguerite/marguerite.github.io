## 为 Discourse 开发一个 Onebox 插件（二）从零开始写 custom engine

上一篇[为 Discourse 开发一个 Onebox 插件（一）理解 Onebox gem](http://marguerite.su/posts/%E4%B8%BA-discourse-%E5%BC%80%E5%8F%91%E4%B8%80%E4%B8%AA-onebox-%E6%8F%92%E4%BB%B6%E4%B8%80%E7%90%86%E8%A7%A3-onebox-gem/) 里，我们知道了 Onebox 是什么样的结构。这篇我们先来写一个 onebox 的自定义引擎。

我们先准备一个脚手架 openbuildservice_onebox.rb

```ruby
module Onebox
  module Engine
    class OpenBuildServiceOnebox
      include Engine
      include LayoutSupport
      include HTML
      always_https

      matches_regexp(%r{^(https?://)?build.opensuse.org/\w+/show/(.)+$})

      private

    end
  end
end
```

我们已经知道了这两个 `module Onebox`和 `module Engine` 嵌套 `class OpenBuildServiceOnebox` 是为什么，为了让 `Preview.new` 的 `ordered_engines`能够找到我们这个以 Onebox 结尾的类，并调用它通过 `matches_regexp` 设置的 `@@matcher`来与真正的 URI 对比来确定唯一一个 engine。`include Engine`的作用是为了得到 `ClassMethods`里面定义的与 URI 做相等比较的方法。另外 `OpenBuildServiceOnebox` 这个 class 没有 `initialize` 方法是因为 `module Engine`里面已经统一实现了，我们可以直接用 `@options`和 `@uri` 这样的实例变量。

但是有一个问题是不是我忽略了？没有 `to_html` 这个最终把 URL 转成 html preview 的函数呀！

不要慌，它实际上是通过 `include LayoutSupport` 实现的：

```ruby
module Onebox
  module LayoutSupport

    def self.max_text
      500
    end

    def layout
      @layout ||= Layout.new(self.class.onebox_name, data)
    end

    def to_html
      layout.to_html
    end
  end
end
```

这个简单的不能再简单的 module 的作用是什么呢？[lib/onebox/layout.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/layout.rb) 和 [lib/onebox/template_support.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/template_support.rb) 可以得到解释：就是去 templates 文件夹下找到 openbuildservice.mustache，用 data 这个 hash 补全，然后 render 成 html。这个 data 自然是返回 hash 的函数，至于返回 hash 的 key 是来自于你的 openbuildservice.mustache 模板。 但也有一些 reserved names 比如 link title favicon image，可以看 layout.rb 的 details 函数。

而 `include HTML` 的作用是这样的：

```ruby
module Onebox
  module Engine
    module HTML
      private
      def raw
        @raw ||= Onebox::Helpers.fetch_html_doc(url, http_params)
      end
    end
  end
end
```

我们可以得到一个名为 raw 的方法，返回使用 rubygem nokogiri 处理过的 URI，你可以从里面挑一些文本来填充你的 data 函数的 hash 的值。

下面我们来写我们的 openbuildservice.mustache 模板。mustache 语法在[这里](https://mustache.github.io/mustache.5.html)。

```html
{{#image}}<img src='{{image}}' class='thumbnail'/>{{/image}}
<h4><a href='{{link}}' target='_blank'>{{title}}</a></h4>
<p>{{description}}</p>
{{#request}}
<p>Submit package <a href='{{source_prj_link}}'>{{source_prj}}</a> / <a href='{{source_pkg_link}}'>{{source_pkg}}</a> to package <a href='{{dest_prj_link}}'>{{dest_prj}}</a> / <a href='{{dest_pkg_link}}'>{{dest_pkg}}</a></p>
<p>
<img />
Created by <a href='{{author_link}}' target='_blank'>{{author_name}}</a>
<span>{{fuzzy_time}}</span>
</p>
<p>
<img />
In state
<a href='{{link}}#request_history'>{{request_state}}</a>
</p>
{{/request}}
<ul>
{{#packages}}
<li>
<ul>
<li class="obs-buildstatus">
<a href='{{repo_uri}}' target='_blank'>{{repo}}</a>
</li>
<li class="obs-buildstatus">
{{arch}}
</li>
<li class="obs-buildstatus">
<a class="{{status_class}}" href='{{buildlog}}' target='_blank'>{{buildstatus}}</a>
</li>
</ul>
</li>
{{/packages}}
</ul>
```

这个基本上就是 html。相信比较好理解，build.opensuse.org 上面能够带 show 的链接主要就四种：user, project, package 和 request。其中前两者没什么好说的，就是有 avatar 就显示，没有就不显示呗。request 就显示提交人的头像，同时显示来源和去向。至于贴 pacakge 的，最值得预览的就是它的编译状态。所以后两者用了条件判断。就是 data 给的 hash 里 package/request key 的值不为空就显示这些。同时根据这些我们也就能知道 data 函数需要什么了，填充一下脚手架：

```ruby
module Onebox
  module Engine
    class OpenBuildServiceOnebox
      include Engine
      include LayoutSupport
      include HTML
      always_https

      matches_regexp(%r{^(https?://)?build.opensuse.org/\w+/show/(.)+$})

      private

      def data
        { 
          image: avatar,
          link: link,
          title: title,
          description: user? ? raw.css('#home-username').text : raw.css('#description-text').text,
          request: request,
          packages: package
        }
      end

      def avatar
        if request?
          author_avatar
        elsif user?
          raw.css('.home-avatar').attr('src')
        end
      end

      def title
        if user?
          raw.css('#home-realname').text
        else
          link.gsub(%r{^.*show/}, '')
        end
      end

      def user?
        link =~ %r{/user/}
      end

      def request?
        link =~ %r{/request/}
      end

      def package?
        link =~ %r{/package/}
      end

      def host
        'https://' + URI.parse(link).host
      end

      def author_link
        host + raw.css('.clean_list li a').first['href']
      end

      def author_avatar
        author_html = Nokogiri::HTML(open(author_link))
        author_html.css('.home-avatar').attr('src')
      end

      def request
        return unless request?

        [{
          "author_link": author_link,
          "author_name": File.basename(author_link),
          "fuzzy_time": raw.css('.clean_list li span.fuzzy-time')[0].text,
          "request_state": raw.css('.clean_list li a')[1].text,
          "source_prj_link": host + raw.css('a.project')[0].attr('href'),
          "source_prj": raw.css('a.project')[0].text,
          "source_pkg_link": host + raw.css('a.package')[0].attr('href'),
          "source_pkg": raw.css('a.package')[0].text,
          "dest_prj_link": host + raw.css('a.project')[1].attr('href'),
          "dest_prj": raw.css('a.project')[1].text,
          "dest_pkg_link": host + raw.css('a.package')[1].attr('href'),
          "dest_pkg": raw.css('a.package')[1].text
        }]
      end

      def package
        reload_id = if request?
                      'result_reload_0_0'
                    elsif package?
                      'result_reload__0'
                    end
        return unless reload_id

        buildstatus(reload_id)
      end
    end
  end
end
```

这个 buildstatus 是最闹心的，它是 javascript，用 nokogiri 直接取是没有的。最后没办法我用 watir gem 去点了一下刷新按钮再取就有了，代码如下：

```ruby
      def buildstatus(reload_id)
        browser = Watir::Browser.new(:chrome, chromeOptions: { args: ['--headless', '--window-size=1200x600', '--no-sandbox', '--disable-dev-shm-usage'] })
        browser.goto(link)
        browser.image(id: reload_id).click

        doc = Nokogiri::HTML(browser.html).css('#package-buildstatus')

        elements = doc.xpath('//div[@id="package-buildstatus"]/table/tbody/tr')
        packages = []
        elements.each do |element|
          repo = element.css('.no_border_bottom a')
          arch = element.css('.arch div')
          build = element.css('.buildstatus a')
          repo_uri = repo.empty? ? '' : host + repo.attr('href').text.strip
          repo_text = repo.empty? ? '' : repo.text
          status_class = if build.text == 'unresolvable' || build.text == 'failed'
                           'obs-status-red'
                         elsif build.text == 'succeeded'
                           'obs-status-green'
                         else
                           'obs-status-grey'
                         end
          packages << { "repo_uri": repo_uri, "repo": repo_text, "arch": arch.text.strip, "buildlog": host + build.attr('href').text.strip, "status_class": status_class, "buildstatus": build.text }
        end

        packages
      end
```

这样基本就写好了。onebox 是支持测试的：

```bash
git clone https://github.com/discourse/onebox
cd onebox
把 openbuildservice_onebox.rb 丢到 lib/onebox/engine，openbuildservice.mustache 丢到 templates
/usr/bin/bundler.ruby2.7 exec /usr/bin/rake.ruby2.7 server
```

然后你可以访问网页测试你写的引擎好使不好使。

