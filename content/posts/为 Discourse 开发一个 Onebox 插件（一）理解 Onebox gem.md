---
title: "为 Discourse 开发一个 Onebox 插件（一）理解 Onebox gem"
date: 2020-11-28T14:59:00+08:00
draft: false
---
## 为 Discourse 开发一个 Onebox 插件（一）理解 Onebox gem

我们 [openSUSE 中文论坛](https://forum.suse.org.cn)用的是 [discourse](https://github.com/discourse/discourse)，有一天给用户贴了一个 [OBS](http://build.opensuse.org/) 的链接，突然想到是不是可以让它也能像 github 一样有一个漂亮的预览小窗口 🤓 于是说干就干：[discourse-openbuildservice-onebox](https://github.com/openSUSE-zh/discourse-openbuildservice-onebox) 插件。

以下是教程，由于涉及到目前最大的 Ruby on Rails 程序 discourse，会分成几部分来讲解。第一部分我们来试着理解一下 discourse 出品的 [onebox](https://github.com/discourse/onebox) gem。

Ruby 作为一门脚本语言，**所有对象的方法都可以被重写**。学名叫做 Meta Programming。这是理解 onebox gem 的基础。

我们下面来看 discourse 是怎么使用 onebox gem 的，下面是 [app/models/post_analyzer.rb](https://github.com/discourse/discourse/blob/master/app/models/post_analyzer.rb) 的 cook 函数，这个函数负责把你输入的文字转为 html 保存在 postgresql 数据库，是最基础的函数之一：

```ruby
  def cook(raw, opts = {})
    [...]

    result = Oneboxer.apply(cooked) do |url|
      @onebox_urls << url
      if opts[:invalidate_oneboxes]
        Oneboxer.invalidate(url)
        InlineOneboxer.invalidate(url)
      end
      onebox = Oneboxer.cached_onebox(url)
      @found_oneboxes = true if onebox.present?
      onebox
    end

    cooked = result.to_html if result.changed?
    cooked
  end
```

可以看到 discourse 自己还有一个 Oneboxer 的 wrapper，这里使用了 Oneboxer.apply 和 Oneboxer.cache_onebox 函数。接下来我们接着看 [lib/oneboxer.rb](https://github.com/discourse/discourse/blob/master/lib/oneboxer.rb)：

```ruby
  def self.apply(string_or_doc, extra_paths: nil)
    doc = string_or_doc
    doc = Nokogiri::HTML5::fragment(doc) if doc.is_a?(String)
    changed = false

    each_onebox_link(doc, extra_paths: extra_paths) do |url, element|
      onebox, _ = yield(url, element)

      if onebox
        parsed_onebox = Nokogiri::HTML5::fragment(onebox)
        next unless parsed_onebox.children.count > 0

        if element&.parent&.node_name&.downcase == "p" &&
           element.parent.children.count == 1 &&
           HTML5_BLOCK_ELEMENTS.include?(parsed_onebox.children[0].node_name.downcase)
          element = element.parent
        end

        changed = true
        element.swap parsed_onebox.to_html
      end
    end

    # strip empty <p> elements
    doc.css("p").each do |p|
      if p.children.empty? && doc.children.count > 1
        p.remove
      end
    end

    Result.new(doc, changed)
  end
```

这段其实没啥用，简单解释下就是取到 cooked 这个 html 里需要 onebox 化的 URL node。然后 invalidate 之后取 cached_onebox。cached_onebox 与实际进行 onebox 化里面有太多的判断代码（onebox_raw)，我们只需要关注 Oneboxer 的这个函数就够了：

```ruby
  def self.external_onebox(url)
    Discourse.cache.fetch(onebox_cache_key(url), expires_in: 1.day) do
      fd = FinalDestination.new(url,
                                ignore_redirects: ignore_redirects,
                                ignore_hostnames: blocked_domains,
                                force_get_hosts: force_get_hosts,
                                force_custom_user_agent_hosts: force_custom_user_agent_hosts,
                                preserve_fragment_url_hosts: preserve_fragment_url_hosts)
      uri = fd.resolve

      if fd.status != :resolved
        args = { link: url }
        if fd.status == :invalid_address
          args[:error_message] = I18n.t("errors.onebox.invalid_address", hostname: fd.hostname)
        elsif fd.status_code
          args[:error_message] = I18n.t("errors.onebox.error_response", status_code: fd.status_code)
        end

        error_box = blank_onebox
        error_box[:preview] = preview_error_onebox(args)
        return error_box
      end

      return blank_onebox if uri.blank? || blocked_domains.map { |hostname| uri.hostname.match?(hostname) }.any?

      options = {
        max_width: 695,
        sanitize_config: Onebox::DiscourseOneboxSanitizeConfig::Config::DISCOURSE_ONEBOX,
        allowed_iframe_origins: allowed_iframe_origins,
        hostname: GlobalSetting.hostname,
        facebook_app_access_token: SiteSetting.facebook_app_access_token,
      }

      options[:cookie] = fd.cookie if fd.cookie

      r = Onebox.preview(uri.to_s, options)
      result = { onebox: r.to_s, preview: r&.placeholder_html.to_s }

      # NOTE: Call r.errors after calling placeholder_html
      if r.errors.any?
        missing_attributes = r.errors.keys.map(&:to_s).sort.join(I18n.t("word_connector.comma"))
        error_message = I18n.t("errors.onebox.missing_data", missing_attributes: missing_attributes, count: r.errors.keys.size)
        args = r.data.merge(error_message: error_message)

        if result[:preview].blank?
          result[:preview] = preview_error_onebox(args)
        else
          doc = Nokogiri::HTML5::fragment(result[:preview])
          aside = doc.at('aside')

          if aside
            # Add an error message to the preview that was returned
            error_fragment = preview_error_onebox_fragment(args)
            aside.add_child(error_fragment)
            result[:preview] = doc.to_html
          end
        end
      end

      result
    end
  end
```

这里可以看到：

```ruby
    r = Onebox.preview(uri.to_s, options)
    result = { onebox: r.to_s, preview: r&.placeholder_html.to_s }
```

这段是最主要的，也就是说 Onebox 这个 gem 是通过它的 preview 函数来与 discourse 进行交互的。discourse 代码看到这里就够了，下面我们来看 onebox 代码。

一个 rubygem 最重要的是它的 lib 文件夹。我们先从 [lib/onebox.rb](https://github.com/discourse/onebox/blob/master/lib/onebox.rb) 看起：

```ruby
  def self.preview(url, options = Onebox.options)
    # onebox does not have native caching
    unless Onebox::Helpers.blank?(options[:cache])
      warn "Onebox no longer has inbuilt caching so `cache` option will be ignored."
    end

    Preview.new(url, options)
  end
```

这就是上面的 `Onebox.preview` 的来源。最重要的是 Preview 这个 class。下面我们接着看 [lib/onebox/preview.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/preview.rb)：

```ruby
module Onebox
  class Preview

    # see https://bugs.ruby-lang.org/issues/14688
    client_exception = defined?(Net::HTTPClientException) ? Net::HTTPClientException : Net::HTTPServerException
    WEB_EXCEPTIONS ||= [client_exception, OpenURI::HTTPError, Timeout::Error, Net::HTTPError, Errno::ECONNREFUSED]

    def initialize(link, options = Onebox.options)
      @url = link
      @options = options.dup

      allowed_origins = @options[:allowed_iframe_origins] || Onebox::Engine.all_iframe_origins
      @options[:allowed_iframe_regexes] = Engine.origins_to_regexes(allowed_origins)

      @engine_class = Matcher.new(@url, @options).oneboxed
    end
  end
end
```

可以看到，这段代码的意思是说根据 Matcher 来匹配 URL 从而取 oneboxed 的结果。接着看 [lib/onebox/matcher.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/matcher.rb)：

```ruby
module Onebox
  class Matcher
    def initialize(link, options = {})
      @url = link
      @options = options
    end

    def ordered_engines
      @ordered_engines ||= Engine.engines.sort_by do |e|
        e.respond_to?(:priority) ? e.priority : 100
      end
    end

    def oneboxed
      uri = URI(@url)
      return unless uri.port.nil? || Onebox.options.allowed_ports.include?(uri.port)
      return unless uri.scheme.nil? || Onebox.options.allowed_schemes.include?(uri.scheme)
      ordered_engines.find { |engine| engine === uri && has_allowed_iframe_origins?(engine) }
    rescue URI::InvalidURIError
      nil
    end
  end
end
```

这段代码的意思是从 `ordered_engines` 里找到匹配 uri 的 engine。返回这个 engine。而 `ordered_engines` 来自 `Engine.engines`。我们来看最重要的 [lib/onebox/engine.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/engine.rb)：

```ruby
module Onebox
  module Engine
    def self.included(object)
      object.extend(ClassMethods)
    end

    def self.engines
      constants.select do |constant|
        constant.to_s =~ /Onebox$/
      end.map(&method(:const_get))
    end
  end
end

require_relative "helpers"
require_relative "layout_support"
require_relative "file_type_finder"
require_relative "engine/standard_embed"
require_relative "engine/html"
require_relative "engine/json"
require_relative "engine/amazon_onebox"
require_relative "engine/github_issue_onebox"
require_relative "engine/github_blob_onebox"
[...]
```

这里的 included 函数其实是一个被重写的内置函数。它是 ruby 的 module 函数，意思是这个 module 被 included 到别的 Object 里的时候会做什么。也就是说 `module Engine` 这个模块被别的 Class/Module 这样 include 的时候会发生什么：

```ruby
module A
    include Engine
end
```

这里所有 include 了 module Engine 的 Class 都会自动获得 ClassMethods 的类方法，ClassMethods 模块在后面一点的地方：

```ruby
    module ClassMethods
      def ===(other)
        if other.kind_of?(URI)
          !!(other.to_s =~ class_variable_get(:@@matcher))
        else
          super
        end
      end

      def priority
        100
      end

      def matches_regexp(r)
        class_variable_set :@@matcher, r
      end

      def requires_iframe_origins(*origins)
        class_variable_set :@@iframe_origins, origins
      end

      def iframe_origins
        class_variable_defined?(:@@iframe_origins) ? class_variable_get(:@@iframe_origins) : []
      end

      # calculates a name for onebox using the class name of engine
      def onebox_name
        name.split("::").last.downcase.gsub(/onebox/, "")
      end

      def always_https
        @https = true
      end

      def always_https?
        @https
      end
    end
```

举个例子：

```ruby
module Onebox
  module Engine
    class YoukuOnebox
      include Engine
      include HTML

      matches_regexp(/^(https?:\/\/)?([\da-z\.-]+)(youku.com\/)(.)+\/?$/)
      requires_iframe_origins "https://player.youku.com"
```

这是 [lib/onebox/engine/youku_onebox.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/engine/youku_onebox.rb)。因为 `include Engine`，所以自动获得 matches_regexp 这个类方法，所以可以调用：

```ruby
matches_regexp(/^(https?:\/\/)?([\da-z\.-]+)(youku.com\/)(.)+\/?$/)
```

来将 YoukuOnebox 这个 Class 的类全局变量 `@@matcher` 设置为上面的 regexp。通过这个例子我们应该大概可以明白为什么能够刚定义一个新类就拥有那么多的类方法了。同样功效的还有 `include HTML`，`include JSON`等等。下一篇用的的时候再给大家看代码。

下面来看第二个方法：

```ruby
    def self.engines
      constants.select do |constant|
        constant.to_s =~ /Onebox$/
      end.map(&method(:const_get))
    end
```

很多新手这时候就无法理解了。[constants](https://apidock.com/ruby/Module/constants) 是什么？它其实 ruby 的一个 Module 方法。返回这个 module 的所有常量，定义在 Module 内部的 Class 也是常量，给段测试代码：

```ruby
module A
  class B
  end
end

module C
 include A
 def self.test
   p constants
 end
end

C.test
```

上面运行 C.test 的结果是 `[:B]`。类名 B 是 C 的常量。那么 `self.engines` 这个函数其实就好理解了。从 Engine module 的全部类中找到以 `Onebox` 结尾的，并且返回一个字符串数组。上面 Youku 的那个例子我们已经看到了，所有的自定义 Engine 都是这个结构：

```ruby
module Onebox
  module Engine
    class YoukuOnebox
```

目的就是让 `self.engines` 函数能够知道它的存在。

我们接着往下看：

```ruby
    def initialize(link, timeout = nil)
      @errors = {}
      @options = DEFAULT
      class_name = self.class.name.split("::").last.to_s

      # Set the engine options extracted from global options.
      self.options = Onebox.options[class_name] || {}

      @url = link
      @uri = URI(link)
      if always_https?
        @uri.scheme = 'https'
        @url = @uri.to_s
      end
      @timeout = timeout || Onebox.options.timeout
    end
```

是不是很有意思，一个 Module 居然有 initialize 函数！他的作用是，include 它的 class 的 new instance 生成的时候，如果这个 class 本身没有写 initialize 函数，那么运行 module 带的。要是写了，在 class 的 initialize 函数里可以运行 super 来运行 module 带的。我们看 engine 文件夹里所有的自定义引擎都没有写自己的 initialize 函数，就是因为这里写了。

这里定义了一些实例变量，这些变量是写自定义引擎的时候可以直接用的。

接着往下看：

```ruby
    # raises error if not defined in onebox engine.
    # This is the output method for an engine.
    def to_html
      fail NoMethodError, "Engines need to implement this method"
    end
```

这种函数看起来很奇怪对不对？直接抛错误。因为 Ruby 是可以重写方法的。上面 YoukuOnebox 的例子里：

```ruby
      def to_html
        <<~HTML
          <iframe src="https://player.youku.com/embed/#{video_id}"
                  width="640"
                  height="430"
                  frameborder='0'
                  allowfullscreen>
          </iframe>
        HTML
      end
```

 就重写了这个方法。这样 Engine module 调用方法的时候实际上是不空的。下面我们来捋一下 `Onebox.preview(url, options)` 的流程：

`Onebox.preview` 调用`Preview.new(url,options)`,后者调用 `Matcher.new(@url, @options).oneboxed`, Matcher 是获取全部自定义引擎后通过判断 `engine == uri`来找出对应引擎。

上面的理解了我们就可以接着理解 `engine == uri` 了。我们前面已经说了，所有的自定义引擎都 `include Engine`了，得到了`ClassMethods`的类方法，有一个类方法就是这个：

```ruby
      def ===(other)
        if other.kind_of?(URI)
          !!(other.to_s =~ class_variable_get(:@@matcher))
        else
          super
        end
      end
```

 如果是拿 URI 来判断的话，先把它转成字符串，然后与自定义引擎的类变量 `@@matcher` 做正则匹配，就能够确保引擎的唯一性。

上面也说了，Onebox.preview 最终得到的结果是 Engine module 里面的一个自定义 Class。那么 discourse 是怎么使用这个 engine 的呢？

```ruby
    r = Onebox.preview(uri.to_s, options)
    result = { onebox: r.to_s, preview: r&.placeholder_html.to_s }
```

我们再把 discourse 的这段代码拿回来理解。它定义了一个 hash。其中 onebox 是这个自定义 class 的 to_s，引擎怎么转字符串呢？在 [lib/onebox/preview.rb](https://github.com/discourse/onebox/blob/master/lib/onebox/preview.rb) 里面：

```ruby
    def to_s
      return "" unless engine
      sanitize process_html engine_html
    rescue *WEB_EXCEPTIONS
      ""
    end
```

对我们用处不大，略。那么 preview 是什么呢？是调用了引擎的 placeholder_html 方法，这个方法在 engine.rb 里面。

```ruby
    def placeholder_html
      to_html
    end
```

它调用了默认抛错误的那个 to_html 方法，而当有了确定的引擎后，to_html 方法是自定义引擎实现的。也就是说，想要写一个自定义引擎，最最重要的方法是 `matches_regexp`和`to_html`。

对于 Onebox gem 的理解到这里就可以了。下一篇我们将要真正开始写我们的插件。

