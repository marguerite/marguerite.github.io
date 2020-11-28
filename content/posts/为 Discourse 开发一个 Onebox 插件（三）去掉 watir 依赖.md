---
title: "为 Discourse 开发一个 Onebox 插件（三）去掉 watir 依赖"
date: 2020-11-28T16:07:00+08:00
draft: false
---

我们在前两篇文章中已经基本实现了这个 engine，但是目前有一个非常恼人的依赖问题：由于 build.opensuse.org 的 package build status 是使用 javascript 加载的，而 nokogiri gem 并不支持 javascript，这就造成了我们需要使用 watir gem 去点一下网页上的 refresh 按钮，才能获取正确的 build status：

```ruby
    browser = Watir::Browser.new(:chrome, chromeOptions: { args: ['--headless', '--window-size=1200x600', '--no-sandbox', '--disable-dev-shm-usage'] })
    browser.goto(link)
    browser.image(id: reload_id).click
```

但是随之带来的依赖是非常恐怖的，我们需要一个 chrome-driver 和一个 chromium。要知道 discourse 没有装在本地的，都是装在 VPS 上面，一个 chromium 带来的空间和内存使用是十分恐怖的。于是我们通过这篇文章来教你如何干掉 watir 依赖。

我们测试用的界面是 [marketo](https://build.opensuse.org/package/show/home:MargueriteSu/marketo) package，使用 chromium 右键查看网页源代码，我们发现 Refresh 按钮是这样的：

```html
    <div accesskey='r' class='btn btn-outline-primary build-refresh float-right' onclick='updateBuildResult(&#39;&#39;)' title='Refresh Build Results'>
      Refresh
      <i class='fas fa-sync-alt' id='build-reload'></i>
    </div>
```

点击它的时候执行的是“updateBuildResult('')”这个 javascript 函数。对于现代网页开发而言，基本上 javascript 都写在一个文件里然后在 html 中引用的，我们翻遍了网页源代码只发现了一个引用：

```html
    <script src="/assets/webui/application-ab8f02f997c1b9a61cc597f3012f2ad83e4a8e135329131cb4df389335ad4ec1.js"></script>
```

使用 chromium 的“检查”功能打开调试器，通过 Sources 标签页可以打开这个 js 文件，自动 pretty 一下后是这样的：

```javascript
function updateBuildResult(t) {
    var i = []
      , o = {};
    $(".result div.collapse:not(.show)").map(function(e, t) {
        var n = $(t).data("main") ? $(t).data("main") : "project";
        o[n] === undefined && (o[n] = []),
        $(t).data("repository") === undefined ? i.push(n) : o[n].push($(t).data("repository"))
    });
    var e = $("#buildresult" + t + "-box").data();
    e.show_all = $("#show_all_" + t).is(":checked"),
    e.collapsedPackages = i,
    e.collapsedRepositories = o,
    $("#build" + t + "-reload").addClass("fa-spin"),
    $.ajax({
        url: $("#buildresult" + t + "-urls").data("buildresultUrl"),
        data: e,
        success: function(e) {
            $("#build" + t + " .result").html(e)
        },
        error: function() {
            $("#build" + t + " .result").html("<p>No build results available</p>")
        },
        complete: function() {
            $("#build" + t + "-reload").removeClass("fa-spin"),
            initializePopovers('[data-toggle="popover"]')
        }
    })
}
```

可以看到这里使用了一个 jquery 的 ajax 请求，访问 url，传递 e 这个 data，就会得到 build status。

我们这时候就想是不是可以通过 ruby 去模拟这个 ajax 请求来直接获得数据，就不需要使用 watir gem 去点一下按钮了。说干就干：

url 要的是网页上 id=“buildresult-urls” 的东西的 buildresultUrl 字段，我们找一下：

```html
    <div class='bg-light' data-buildresult-url='/package/buildresult' id='buildresult-urls'>
    <ul class='nav nav-tabs pt-2 px-3 flex-nowrap' data-index='' data-package='marketo' data-project='home:MargueriteSu' id='buildresult-box' role='tablist'>
```

是 `/package/buildresult`。而需要传的值是 e 也就是 buildresult-box 里的 data。看到是 index, package, project 的内容。另外我们还看到了 collapsedPackages 和 collapsedRespositories 的内容，一个是数组 i 一个是散列 o，内容分别是 `<div class="result"></div>` 下面的 `<div class="collapsed"` 里的 `data-main` 和 `data-repository` 项。但网页源代码里 `<div class="result">` 是空的。

我想这跟 `updateBuildResult()` 这个函数需要面对的情景有关的，我们现在的情景是 div result 是空的，而它在按钮被按下过一次后是有值的。反正 Get 请求没有参数都可以，我们就先来写一下试一试也无妨：

```ruby
require "net/http"
require "uri"

url = 'https://build.opensuse.org/package/buildresult?index=&package=marketo&project=home%3AMargueriteSu'
uri = URI(url)

resp = Net::HTTP.get_response(uri)
puts resp.body
```

结果返回了 404。一想也对嘛，毕竟 openSUSE 的 web developer 是 darix, 不可能不验证嘛。但是 package build status 是不需要登录 openSUSE 账户就能看见的，那么验证的方式应该就是 cookie 了，我们在 chriomium 网页调试器的 Netowrk 界面点开请求看一下是这样的：

```yaml
:authority: build.opensuse.org
:method: GET
:path: /package/buildresult?project=home%3AMargueriteSu&package=marketo&index=&show_all=false
:scheme: https
accept: */*
accept-encoding: gzip, deflate, br
accept-language: zh-CN,zh;q=0.9,en;q=0.8,fr;q=0.7,ja;q=0.6,zh-TW;q=0.5
cookie: _obs_api_session=Wofeza%2B6Orv1yBRudIdkuaVk2%2FrGU3aawTLraZjoBHDZgiAghLJ6Rty8%2F6LkYJq35V1X9AewWVteQaKjbc8bkBz%2BCXVgvnw1LCHTLENEcJ%2BOrDZduKxogn92p8JtGkrsAg2eCFNIf9PlHqZUO3VRxsQCCchW0Yctf%2FVKS5Dt9w%2FFK4MuNgcepd%2F9efbaMGuD8tPXH6BswiOb6AhF9cMvWB09UiQM0jB5Pw0J6L0x6hTOYNigz7%2BFc1yJxlgMK%2B39RQCEBGUPuvB8npwzovKq9I0ufkQYooTln0GX4trsfpjq4xN4JgGo1WxUlcVurN4AOfg%2FFw%3D%3D--6GcZFXO4PpWG7EZU--HxRMQH1UYEi16AZFmLSEnw%3D%3D
if-none-match: W/"f218b807b90379d9cf5bbc186b1f0310"
referer: https://build.opensuse.org/package/show/home:MargueriteSu/marketo
sec-fetch-dest: empty
sec-fetch-mode: cors
sec-fetch-site: same-origin
user-agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36
x-csrf-token: iSzEuQG8YQKZwuCk7Ka5dhIJjhKmSyCIckRXjmjqt+IKhO0R4x3L6r639vSiF0RA3KOMVedCT+zBUp0JIK/xNg==
x-requested-with: XMLHttpRequest
```

好么，不只是 cookie 里有 token，还有 x-csrf-token。验证一般就是验证 User Agent 是不是爬虫，cookie 内容，再就是如果是 ajax 请求，我们用普通的 GET 可能也不行，服务器可能会查 x-requested-with 甚至是 referer。

cookie 我们可以通过先访问一下网页来得到，但是 csrf-token 怎么得到呢？网页源代码里面有：

```html
<meta name="csrf-token" content="XJTaLoLQsp6EyezGStiZgW2O5FectHPabI+hcnEdT/jfPPOGYHEYdqO8+pYEaWS3oyTmEN29HL7fmWv1OVgJLA==" />
```

同样这些 index, package, project 也都在网页源代码里，show_all 就一直让它为 false 好了。于是我们写出了第二版：

```ruby
require "net/http"
require "uri"
require "nokogiri"

url = 'https://build.opensuse.org/package/show/home:MargueriteSu/marketo'
uri = URI(url)

resp = Net::HTTP.get_response(uri)

cookie = resp['Set-Cookie'] // 获取 cookie

doc = Nokogiri::HTML(resp.body)

token = doc.at("meta[name='csrf-token']")['content'] // 抓 csrf token
path = doc.at("div[id='buildresult-urls']")['data-buildresult-url'] // 抓 path
index = doc.at("ul[id='buildresult-box']")['data-index']
package = doc.at("ul[id='buildresult-box']")['data-package']
project = doc.at("ul[id='buildresult-box']")['data-project']

query = Hash["index"=>index,"package"=>package,"project"=>project,"show_all"=>false] // 造一个 uri 的 query
new_uri = uri
new_uri.path = path
new_uri.query = URI.encode_www_form(query)

http = Net::HTTP.new(new_uri.host, new_uri.port)
http.use_ssl = new_uri.scheme == 'https'
request = Net::HTTP::Get.new(new_uri)
request['Cookie'] = cookie
request['X-Requested-With'] = 'XMLHttpRequest'
request['X-CSRF-Token'] = token
request['Referer'] = uri.to_s
request['User-Agent'] = 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36' // 从上面扒的 UA

response = http.request(request)
puts response.body
```

运行一下，结果得到了正确的页面。我们这么直接 puts 肯定不行，需要一个数据结构的嘛：

```ruby
require 'json'
new_doc = Nokogiri::HTML(response.body)

buildstatus = new_doc.at("div[id='package-buildstatus']")
repositories = buildstatus.css("div[class*='show']")

buildresult = Hash.new
repositories.each do |repository|
  target = repository['data-repository']
  arch = repository.css(".repository-state").text().strip!
  state = repository.css(".build-state").text().strip!
  h = Hash["arch"=>arch, "state"=>state]
  if buildresult[target].nil?
    buildresult[target] = Array[h]
  else
    buildresult[target].append(h)
  end
end

puts JSON.pretty_generate(buildresult)
```

于是就得到了一个散列结构和很好看的输出：

```json
{
  "openSUSE_Tumbleweed": [
    {
      "arch": "x86_64",
      "state": "succeeded"
    },
    {
      "arch": "i586",
      "state": "unresolvable"
    }
  ]
}
```

到了这里，我们只需要把这些测试代码变成一个 Class 写到 `engine/openbuildservice_onebox.rb` 里面去，就可以删掉 buildstatus 函数了。最终的结果见：[discourse-openbuildservice-onebox](https://github.com/openSUSE-zh/discourse-openbuildservice-onebox)
