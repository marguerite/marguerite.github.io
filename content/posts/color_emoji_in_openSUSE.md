---
title: "Color Emoji in openSUSE"
date: 2020-11-04T00:00:00+08:00
draft: false
---
上一篇文章里我们讲了 fontconfig 常见的几个坑，今天我们来继续讲一讲 openSUSE 的 Colored Emoji 支持。也就是如何配置 Noto Color Emoji 这个字体用于网页显示（用于终端显示是另一回事，涉及到比如 vte 的 teminal 之类的，有几个相关的 bug 涉及到比如宽度之类的；GTK/Qt 显示又是另一回事，涉及到 cairo）。

为什么是这个字体呢？我也很无奈啊…Noto 系列是 openSUSE 的默认字体，可以说除了英文 locale 别的都是 Noto 来显示的，Noto Color Emoji 跟其它 Noto 字体的 metrics 兼容。这一点就秒杀了其它 Emoji 字体。再者 Emoji 字体本身就不多，目前为止也就 45-generic.conf 里列出的那么不到十个，EmojiOne Color 因为版权问题不再开发了，真正 Linux 上能用的 Colored Emoji 也就剩下一个 Twitter Color Emoji 了。剩下的要么专有的要么没有颜色。

我们先来回忆一下之前的说法：

除了比如 Unicode Full Emoji List 这种专门用于测试 emoji 显示的 URL，大部分我们常见的网页在 css 里是不写 emoji 字体的。

据我摸索的经验，字体的匹配分为三种场景

第一种是直接去匹配这个字体，fc-match “Noto Color Emoji” 这样，也就是 css 的 font-family 里直接写了这个字体。

这部分的支持很简单，比如它要 Apple Color Emoji 或者 Segoe UI Emoji 我没有，可以写个 alias：

    <alias>
      <family>Apple Color Emoji</family>
      <accept><family>Noto Color Emoji</family></accept>
    </alias>

第二种是通过字族匹配，fc-match “emoji” 这样，会从这个字族的 pattern 里找（emoji，symbol，math，fantasy 这四个字族与 sans-serif，serif 和 monospace 这三个主要关注的字族还有所不同，它们的 donePattern 后面还有 sans-serif 字体，因为 49-sansserif.conf）

这部分的支持也很简单，fontconfig 已经默认把 emoji 字体的 default family 全设置成 emoji 了，我们只需要通过 prefer 来实现发行版默认的 Colored Emoji 字体就行了，如果默认的是 Noto Color Emoji 那么什么都不用做。

第三种是字符级别的（glyph），比如你所有的 sans-serif 字体里都没有这个 unicode codepoint 对应的 glyph。这个时候 pattern match 其实已经没用了，因为已经 out of range 了，所有的 sans-serif 都没有，那 sans-serif 的 pattern 也就没用了。它用的是 fc-scan 到的 unordered list，也就是在你系统里找有这个 glyph 的字体来显示。

第三种场景是不经常遇到的，是一种最终的 fallback。但是在面对 emoji 的时候，它是出现次数最多的场景。因为绝大部分网页不会写特定的 emoji 字体也不会写 emoji 字族。

其实针对第三种场景，fontconfig 是提供了解决方案的，就是把字体 prefer 到 pattern 的前面。很多网上能够搜索到的方案也都是这么做的，比如 EmojiOne Color 和现在的 Twitter Color Emoji。

推出我们的方案之前，先来说说网上的方案。大部分是把 emoji pattern 直接扔到 sans-serif，serif 和 monospace 前面。于是带来了一个问题，就是如果 emoji 字体中不只是 emoji，就会用 emoji 字体显示非 emoji 内容。这里说的不限于比如 emoji 字体里有个 A，你所有的 A 都被替换了，还包括使用 emoji 字体的字符宽度来显示 fallback 字体的字符。比如 Noto Color Emoji 作为 emoji 字体，字符宽度特别大，造成了空格的宽度也特别大，在 chromium 中，即使实际的笔画都是用 Libration Sans 显示的，空格宽度不是。你就看到了超长空格的效果。这个也许可以通过指定 emoji 字体的字宽上限来解决，但是目前没有发现谁去认真的解决这个问题。

emoji 字体本来就是作为 Fallback 字体存在的，理论上不应该作为第一个字体出现。为了避免上述情况，现行的解决方案是先把 emoji 扔到前面，再把一个完全没有 emoji 的字体再扔到前面。但这么做的后果就是你的默认英文字体被替换了。而且在 CJK 中问题更加严重，我们本来就是通过把 CJK 字体扔到前面，再把英文字体再扔到前面（有的发行版省略了这步，导致实际上显示英文的不是发行版选定的英文字体，而是默认 CJK 字体中自带的英文部分）来实现中文显示的，这时候要多扔一个 emoji 字体，还要多扔一个 emojiless 的英文字体。扔的顺序很重要，导致好多按数字排列的 fontconfig conf 都要重构。

我解决这个问题的思路不是去使用 prefer 这个严重依赖顺序而且脆弱的方案。我的思路是让出现在我选的 Emoji 字体之前的所有字体都是 emojiless 的。这样就不会破坏发行版默认的字体。

我使用的是一个小众的方法，就是在 scan 的时候从已有字体中屏蔽特定字符集（charsets）。这个方法只有一个 RedHat 的 [bug#31969](https://bugs.freedesktop.org/show_bug.cgi?id=31969) 提到过，但是也没提过怎么看成功没成功，我把它摸索成熟了，下面给大家讲一讲：

<match target=“scan”>
    <test name=“family”>
        <string>Noto Color Emoji</string>
    </test>
    <edit name=“charset” mode=“assign”>
        <minus>
            <name>charset</name>
            <charset>
                <int>0x20</int>
                <range>
                    <int>0x21</int>
                    <int>0x22</int>
                </range>
            </charset>
        </minus>
    </edit>
</match>

以上就是方法，charset 支持 int 和 range。这里的 int 不是 interger 类型，实际上是个 Hex 类型 233，range 的两个字符是起止（包含起止）。

然后在 FC_DEBUG=4 fc-match “Noto Color Emoji” 就能看到这样的 debug 输出：

    Add SubSt match
    [test]
         pattern any family Equal “Noto Color Emoji”
    [edit]
         Edit family Assign Minus charset;

但是除此之外什么都没有...于是我一直以为它没有生效。实际上需要这么测试：

先做一个 local fonts.conf 把 Noto Color Emoji prepend_first：

    <match>
      <edit name=“family” mode=“prepend_first”>
        <string>Noto Color Emoji</string>
      </edit>
    </match>

然后通过：

    FC_DEBUG=4 pango-view -q -t '🧀' 2>&1 | grep -o 'family: "[^"]\+' | cut -c 10- | tail -n 1

来看，正常这个 cheese 表情是 Noto Color Emoji 显示的，但是屏蔽了之后就不是了，说明屏蔽成功。

(说点题外话，很多人不知道那个表情怎么打上去，可以通过 fcitx 的 unicode 输入模式实现，就是同时按 ctrl + alt + shift + space 四键，然后输入 0x 的 unicode codepoint，最后按 Alt + 数字上屏）

而 RedHat 的维护者也给了我一个更好的方案：

    fc-list “Noto Color Emoji”:charset=0x20

如果字体未安装或屏蔽成功，就没有结果，不然会返回字体文件和字体名称。

有了方法剩下就是操作的问题了，但是操作其实比方法更复杂。

我们要先统一下概念。fontconfig 里用的叫做 charset（字符集），而我前面说过一个 unicode codepoint。unicode point 是一个码点，比如 A 字母 1 数字都是一个独立的码点，但是不是每个码点在现实中都能找到对应的字符。字体制作是针对 unicode point 来的。unicode codepoint 的表示法在 fontforge 里叫 U+20，在 fontconfig 的非 debug 输出里叫 20，而在写配置的时候又要用 0x20，这三个都是等价的，但是在不同的程序里必须用不同的表示法。charset 就是码点的集合。

    fc-scan —format “%{charset}” NotoColorEmoji.ttf

就能够看到某个字体有哪些码点。

但是 Emoji 其实有个特殊的地方，可以看 emojipedia 的这篇文章来了解。就是有的时候显示出来的是一个 emoji，但实际上背后是三个甚至四个码点组成的 sequence。我们在测试 emoji 字体的 coverage 的时候，不能随便的取舍，因为把 modifier codepoint 给屏蔽了会造成好多 emoji 都无法显示。

我关于操作是这么构思的，首先我把想要使用的 Emoji 字体的 charset 找出来，转成 unicode codepoints（另一种思路是找 unicode 11 full emoji list 的 data file，这个文件能找到但是我看不懂，而且有重复出现的码点解析起来费劲，我就直接去网上爬演示的 html，再把 emoji 转成 unicode points，这个思路可以看我的 analyze-noto-color-emoji.rb），然后简单处理一下（主要是 Noto Color Emoji 还有空格数字和一些常见 symbol，比如有些在线播放器会用的▶️符号，这个用彩色显示播放器样式会出问题），再拿目标字体的 charset 去取交集，这样得到的是两个字体里都存在的 unicode point，说明这个字体提供了 emoji，最终我写出来了test_emoji_coverage.rb。

剩下就是大规模应用了，我写了一个针对所有 /usr/share/fonts/truetype 文件夹下的字体文件，扫描出 emoji 部分并自动生成 fontconfig blacklist 文件的代码generate_81_emoji_blacklist_glyphs.rb。目前 fonts-config 包里是我用我的 openSUSE 生成的。但未来会想办法用 perl/go/c 之类的重写下，然后做成安装包后自动针对用户的系统生成对应的配置文件。

至此，出现在 Noto Color Emoji 前的字体，或者说系统中除了 Noto Color Emoji 别的字体都不会有 emoji 了。下面是测试：

测试 emoji 字体要有一个测试页面。我主要是用 chromium 浏览器来访问 EmojiOne Color 的测试页面，因为 Firefox 已经捆绑了自己的 Twemoji Mozilla 字体，它的 emoji 显示是没有问题的。

一访问还是不行？

原来是我在生成的时候没有针对 Noto Emoji 这个黑白 emoji 字体，这是正确的，因为我要给它也做 blacklist，它的 charset 就被清空了233，那样的效果等于不载入，那就直接不载入好啦：

<match target=“scan”>
    <selectfont>
        <rejectfont>
            <patlet name=“family”>
                <string>Noto Emoji</string>
            </patlet>
        </rejectfont>
    </selectfont>
</match>

注意不能用 prefer work，因为之前说过第三种匹配类型实际上用作匹配的字体不是 pattern 里面的字体，你在把它加入 pattern 前根本改变不了它的顺序。未来会把这部分代码也整合到生成器里面去，这样比如你指定一个 Colored Emoji 字体，会自动屏蔽所有其它 emoji 字体和非 emoji 字体中的 emoji 部分。
