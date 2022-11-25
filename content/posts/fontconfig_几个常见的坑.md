---
title: "fontconfig 几个常见的坑"
date: 2020-11-04T00:00:00+08:00
draft: false
---
最近 Microsoft 加入 OIN，贡献了它的 60000+ 项专利，这使得 openSUSE 的 freetype2 终于能够开启 ClearType 引擎了。之前 infinality 项目贡献了三大块，我们引入了两大块，但是其中第一大块的 ClearType 引擎没有默认开启，第三大块的非专利色彩滤镜也一直没有。现在专利的问题没有了，freetype2 终于更新到完全体了，它现在有两个引擎，第一个引擎是 Adobe 的 CFF 引擎，主要用于 Noto Sans CJK，第二个引擎就是 infinality 贡献的 ClearType 引擎了。总之都是好东西。

后端更新了，我也需要更新 fonts-config 来默认设置 rgba 和 hintstyle。openSUSE 的 fonts-config 是一系列跟 fontconfig 一样的字体配置文件，已经很老了，于是我需要 modernize 它一下。在这个过程中我几乎看了网上能够找到的全部 fontconfig 相关文章。里面大坑套小坑，有必要专门写文来澄清一下：

## 第一个坑：从 monospace 中清除 sans-serif

比如 [Hack 字体的官方配置](https://github.com/source-foundry/Hack/issues/408)，还有最著名的 [eev’s rant about fontconfig](https://eev.ee/blog/2015/05/20/i-stared-into-the-fontconfig-and-the-fontconfig-stared-back-at-me/)，都推荐了这么一种做法：

```xml
<match>
    <test name=“family” compare=“eq”>
        <string>sans-serif</string>
    </test>
    <test name=“family” compare=“eq”>
        <string>monospace</string>
    </test>
    <edit name=“family” mode=“delete”/>
</match>
```

意思是如果 pattern 有 sans-serif，还有 monospace，就把 sans-serif 
删除。目的是让字体只有一个 generic name。起因在于 /usr/share/fontconfig/conf.avail/49-sansserif.conf（针对所有没有 sans-serif、serif 和 monospace 的 pattern 加上 sans-serif）

甚至还有衍生：

```xml
<match>
    <test name=“family” compare=“eq”>
        <string>Hack</string>
    </test>
    <test name=“family” compare=“eq”>
        <string>sans-serif</string>
    </test>
    <edit name=“family” mode=“delete”/>
</match>
```

把 Hack 字体从 sans-serif 字族中删除（因为 Hack 是 monospace 字体）

听起来是不是很美？这样我就可以 monospace 字族只用 monospace 字体，sans-serif 字族只用 sans-serif 字体了。

我面临的现实问题是 Noto Color Emoji 在 sans-serif 中优先级很低，而这个字体的英文部分会把字母之间的间距变得超大，所以不能 prepend_first 到 sans-serif（网上的教程全部都是错的，或者说你以为 EmojiOne Color 可以 Noto Color Emoji 就可以，那是你想当然了）。于是想到的就是保证出现在 Noto Color Emoji 之前的字体全部都没有 emoji，先把 monospace、serif 之类的从 sans-serif 中去除，可以极大的减少工作量。

但实际完全不是这么一回事。要理解原因，我们需要补习一下 fontconfig 的基础知识（当然有些人是新学233）：

fontconfig 是通过三个步骤来匹配到合适的字体的。第一步叫做 scan，就是扫描系统中的字体来构建一个 list（fc-list 可以还原这个过程），这个 list 是无序的。第二步叫 pattern match，就是针对你要匹配的字体名称比如 “sans-serif”去调整上面的 list，各种 prepend。第三步叫 font match，就是针对调整过的 list 去应用诸如 hinting antialias 之类的。

虽然 fontconfig 的配置文件是有数字顺序的，但上面那三个过程是死的，意思是即使你把 `<match target="font">` 写到 `10-*.conf` 里去，它也不会在 `90-*.conf` 的 `<match target="pattern">` 之前执行。所以开发者在 pattern match 阶段只需要关注对应的 pattern 的先后顺序就可以了。

通过 FC_DEBUG=4 fc-match -s monospace 查看（搜索 “Edit family Delete none”），这段是添加规则

```xml
Add SubSt match 
[test]
         pattern any family Equal “sans-serif”
         pattern any family Equal “monospace”
[edit]
         Edit family Delete none;
```

注意这里的 none，不表示没匹配到。它表示的是 <edit name=“family” mode=“delete/> 这个 edit block 里面没有东西，因为 delete 和 delete_all 规则就是这样的。

真正的匹配阶段是类似这样的：

```xml
FcConfigSubstitute test pattern any family Equal “mono”
No match
FcConfigSubstitute test pattern any family Equal “sans serif”
No match
FcConfigSubstitute test pattern any family Equal “sans”
Substitute Edit family Assign “sans-serif”

Append list before “sans”(s) [marker]
Append list after “sans”(s) “sans-serif”(s) 
FcConfigSubstitute editPattern has 3 elts (size 16)
    family: “sans-serif”(s)
    lang: “en”(w)
    prgname: “fc-match”(s)
```

这个是比较完整的规则，还原成 xml 形式就是这样的：

```xml
<match target=“pattern”>
    <test qual=“any” name=“family” compare=“eq”>
        <string>mono</string>
    </test>
    <test qual=“any” name=“family” compare=“eq”>
        <string>sans serif</string>
    </test>
    <test qual=“any” name=“family” compare=“eq”>
        <string>sans</string>
    </test>
    <edit name=“family” mode=“assign”>
        <string>sans-serif</string>
    </edit>
</match>
```

通过几个 No match 和 editPattern 我们可以看出，这段其实匹配到了一个对 sans 的请求，但把它完全替换成了 sans-serif。

上面的懂了，就可以拔高一些了。Edit Delete 的匹配阶段的 Debug 代码是类似这样的：

```xml
<match>
    <test name=“family” compare=“eq”>
        <string>DejaVu Sans</string>
    </test>
    <test name=“family” compare=“eq”>
        <string>sans-serif</string>
    </test>
    <edit name=“family” mode=“delete”/>
</match>

FcConfigSubstitute test pattern any family Equal “monospace”
No match
FcConfigSubstitute test pattern any family Equal “DejaVu Sans”
FcConfigSubstitute test pattern any family Equal “sans-serif”
Substitute Edit family Delete none

FcConfigSubstitute editPattern has 3 elts (size 16)
    family: “Noto Sans”(w) “sans-serif”(w)
    lang: “en”(w)
    prgname: “fc-match”(s)
```

第一，它会自作主张的给你加一段针对 monospace 的测试（原因没有详查）；第二，它没有告诉你删除成功了没。你需要去看它前面一个规则的 editPattern 来自己比较才知道删除成功了没。我这里前面规则的 editPattern 里有 “DejaVu Sans”，但这里的 editPattern 没有了。所以是成功了。

拔高成功了，直接上炼狱模式，去真实的 FC_DEBUG=4 fc-match -s monospace 里面找，会发现只有添加规则的，别的什么都没有找到。这就是 fontconfig interesting 的地方了，如果规则没有应用，它不会告诉你。

但是我会告诉你呀！再来看这条规则：

```xml
<match>
    <test name=“family” compare=“eq”>
        <string>sans-serif</string>
    </test>
    <test name=“family” compare=“eq”>
        <string>monospace</string>
    </test>
    <edit name=“family” mode=“delete”/>
</match>
```

它需要测试 pattern 的 family 字段中有 sans-serif，**并且**有 monospace。但实际上把整个匹配过程从头到尾读过了之后，会发现这样的条件永远也不会满足。之所以有这么一个无用的测试，是对 49-sansserif.conf 的误解造成的：

```xml
<match target="pattern">
    <test qual="all" name="family" compare="not_eq">
        <string>sans-serif</string>
  </test>
  <test qual="all" name="family" compare="not_eq">
	<string>serif</string>
  </test>
  <test qual="all" name="family" compare="not_eq">
	<string>monospace</string>
  </test>
  <edit name="family" mode="append_last">
	<string>sans-serif</string>
  </edit>
</match>
```

这段规则的实际意思是说，如果去匹配一个字体 fc-match -s “我瞎编的字体名”，这时 fontconfig 不知道“我瞎编的字体名”究竟是什么字族，甚至不知道这个字体在系统中有没有，为了防止显示不出来，就把 sans-serif 旗标放在最后，这样后续针对 sans-serif 旗标的各种 prefer 往 list 里加字体，最后用的是 sans-serif 的第一个字体显示。

但是如果是 fc-match -s monospace 呢？不好意思 monospace 已经有了，永远也不会添加 sans-serif 旗标到最后。也就永远也不存在两个字族在一个 list 里的场景。

好，杠精来了，如果我手动把一个字体既 alias 成 sans-serif，又 alias 成 monospace 呢？那也没有影响。下面这个规则：

```xml
<alias>
    <family>Georgia</family>
    <default>
        <family>serif</family>
    </default>
</alias>
```

实际上等于：

```xml
<match target=“pattern”>
    <test name=“family” compare=“eq”>
        <string>serif</string>
    </test>
    <edit name=“family” mode=“append_last”>
        <string>Georgia</string>
    </edit>
</match>
```

就是语法糖而已。alias 后，fc-match -s sans-serif/monospace 肯定都有这个字体，但也只是这样了。

我明白你们的逻辑，alias 是替身的意思，一个字体即是 sans-serif 的替身，又是 monospace 的替身，那么匹配这个字体，list 里面必然又有 sans-serif 又有 monospace。

但这个逻辑是完全没有道理的。绝大多数人都有这样的误区：

认为 sans-serif 字族像 family 一样，是一个子列表，匹配 sans-serif 等于匹配了整个字族的所有字体。append sans-serif 等于把所有的 sans-serif 字体放在后面。这实际上是把 sans-serif/serif/monospace 看成了字体的一个 bool 属性，把现实中对 sans-serif 字体的理解代入到 fontconfig 里面来了。

我很早就说过，没有名为 sans-serif 的字体。在 fontconfig 中，sans-serif 只是一个锚点。这个锚点跟 DejaVu Sans 或 Noto Sans 这些真实字体的地位是相同，唯一的不同就是现实中没有名为 sans-serif 的字体，因此永远也匹配不到，所以才需要各种 append/prepend work，保证在匹配 sans-serif 锚点的时候 family 列表的 sans-serif 之前或之后有别的字体，即使 family 列表中的 sans-serif 永远不会匹配到，也总有字体会匹配到。你们认为的过程是类似这样的：

```xml
family: DemoFont
hintstyle: 1
lang: en
arbitrary_attributeA: sans-serif
arbitrary_attributeB: monospace
```

然后匹配 sans-serif 是去所有字体中找属性。但真实的过程中 family 列表的增长是类似这样的：

```xml
family: sans-serif
family: Noto Sans, sans-serif
family: Noto Sans, sans-serif, DejaVu Sans
lang: en
hintstyle: 1
```

最后程序拿着这个 family 列表要求 sans-serif，第一个 Noto Sans 没安装，第二个永远没有，于是就用第三个 DejaVu Sans。我们可以看出来，无论是 prepend/append 还是 alias，都是针对一个对象去调整它之前之后的字体。这才是 pattern match 的意思。而你们认为的其实是在第二阶段 font match 也就是 <match target="font"> 阶段发生的事情，但这个阶段的 pattern 是已经固定的。

如果把一个字体既 alias 到 monospace 又 alias 到 sans-serif，那么其实是两个对象，sans-serif 和 monospace。但如果对象本身就没在 family 列表中，针对它的 match 就不会应用。所以你去匹配 monospace 的过程是这样的：

```xml
family: monospace
family: monospace, DejaVu Sans Mono
```

前面说过了，有 monospace 不会自动加 sans-serif。sans-serif 对象都没有，针对它的调整肯定也不会应用。而你 fc-match -s “DejaVu Sans Mono” 呢？

会被当作 sans-serif 处理。family 列表的增长是这样：

```xml
family: DejaVu Sans Mono
family: DejaVu Sans Mono, sans-serif
family: DejaVu Sans Mono, Noto Sans, sans-serif
```

这里根本没有 monospace 出现。因为你的 alias 的作用对象是 monospace 而不是 DejaVu Sans Mono。

所以除非你手动的把 sans-serif append 到 monospace 后面，否则不会出现两个虚拟字体在一个 family 列表的情况。

这里再澄清一点：

即使你能够把 sans-serif append 到 monospace 后面，你 append 的也只是 sans-serif 这个虚拟字体本身，而不是所有的 sans-serif 字体。因为这些已知的字体是通过别的 append/prepend 加到 sans-serif 前后的。但是 pattern match 是有先后顺序的，你的 sans-serif append 的晚了，别的 match 已经完事了，你也就只能 append 一个虚拟字体。因此 fontconfig 的出厂配置在 append sans-serif 时才会尽量在 pattern match 的最前面做。

好，杠精又来了，Hack 字体的配置总不会错了吧？如果既有 Hack 又有 sans-serif 就把 Hack 删了。实际上也完全不是这回事。我们用 DejaVu Sans 做个测试：

```bash
$ fc-match -a sans-serif
“DejaVu Sans” “Book”
“DejaVu Math TeX Gyre”
...
```

应用了上面的那个 conf 之后结果是这样的：

```bash
 $ fc-match -a sans-serif
 “DejaVu Math TeX Gyre”
 “DejaVu Sans” “Book”
 ...
```

咦？DejaVu Sans 怎么还有？更 interesting 的是：

```bash
$ fc-match -a “DejaVu Sans”
“DejaVu Sans” “Book”
...
```

这时就需要去理解我说的三个阶段了。要知道，pattern 永不会落在虚处。fontconfig 的 pattern 里面存在了太多不一定存在的字体名称，比如 sans-serif 自己就是。如果直接返回 pattern，就存在整个 family 列表中的字体都没有安装在你的电脑上的场景。

那么 fontconfig 是怎么做的呢？我没有详细看过源代码，但合理猜测是这样：

在 scan 阶段把指定文件夹的全部字体都扫描成一个无序列表。在 pattern 和 font 阶段处理 pattern 和字体渲染属性。在最终匹配阶段，把 pattern 应用到无序列表。根据 pattern 里的项，如果无序列表中没有，没安装的字体就 drop 掉，如果有，就根据 pattern 调整位置，得到微调过的无序列表，而最终顺序是根据算法（family lang style 等有不同的权重）匹配所需字体属性和列表字体提供的属性，再调整微调过的无序列表输出。

那么问题来了，如果是 pattern 中没有的字体呢？只要无序列表中有，一样参与运算，最终结果也有。

这就解释了为什么 fc-match -a sans-serif 时还有 DejaVu Sans 了。虽然规则生效了，从 pattern 中删掉了，但是任何非 scan 阶段的规则都无法改变扫描阶段生成的无序列表。于是它又继续出现了。

而直接 fc-match -a “DejaVu Sans” 它会出现在第一个的原因也一样，它自身参与了运算，那么自己肯定是最匹配自己的。

所以无论怎么调整，只能调整排序。如果不想要某个字体，要么直接删掉，要么在扫描阶段通过：

```xml
<match target=“scan”>
    <selectfont>
        <rejectfont>
            <patlet name=“family”>
                <string>DejaVu Sans</string>
            </patlet>
        </rejectfont>
    </selectfont>
</match>
```

来做。

## 第二个坑：禁用 Mozilla Firefox 自带的 Twemoji Mozilla 这个 emoji 字体

Firefox 自己捆绑了一个 emoji 字体，并且在源代码里面把它作为默认的 emoji 字体。而 Linux 上面的 emoji 字体大部分是 Noto Color Emoji。为了统一风格，我想要让 firefox 匹配字体时不匹配它：

```xml
<match>
    <test name=“family” compare=“eq”>
        <string>Twemoji Mozilla</string>
    </test>
    <test name=“prgname” compare=“eq”>
        <string>Firefox</string>
    </test>
    <edit name=“family” mode=“delete”/>
</match>
```

看似完美，但实际上不能用，道理跟上面讲的一样。因为现在网页字体在 css 里不会在最前面指定 emoji 字体：

```css
font-family: “emoji”, “sans-serif”;
```

做的差的直接就是 sans-serif，做的好的用的也都是真正的 emoji 字体名，比如：

```css
font-family: “Apple Color Emoji”, “sans-serif”;
```

主要是 fontconfig 的 emoji 标准（lang=“und-zsye” 和 “emoji” 这个 generic name）定晚了。

针对 Apple Color Emoji，我们可以用 alias 来把 Noto Color Emoji append 到后面，这样原装字体没有就会用我们的李鬼字体。但是绝大多数时候我们的 emoji 其实是在跟 sans-serif 在争。

根据我们在第一个坑里面讲的原理，即使把 Twemoji Mozilla 从 pattern 里面删了也没用，它还是会出现在 sans-serif 中。所以要么把它整个 reject 掉，要么把 Noto Color Emoji 扔到它的前面：

```xml
<match>
    <test name=“family” compare=“eq”>
        <string>Twemoji Mozilla</string>
    </test>
    <test name=“prgname” compare=“eq”>
        <string>firefox</string>
    </test>
    <edit name=“family” mode=“prepend”>
        <string>Noto Color Emoji</string>
    </edit>
</match>
```

这样的 block 可以有多个，最终保证无论它要哪一种 emoji 字体，不管安装没安装，第一个匹配到的字体都只会是 Noto Color Emoji。

未完待续。
