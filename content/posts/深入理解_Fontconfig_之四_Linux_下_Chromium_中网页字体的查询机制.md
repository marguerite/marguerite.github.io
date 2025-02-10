---
title: "深入理解 Fontconfig 之四：Linux 下 Chromium 中网页字体的查询机制"
date: 2025-02-10T00:00:00+08:00
draft: false
---

很多人包括我，都是从这个官方说明入坑的：[Blink's Text Stack](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/platform/fonts/README.md)。

Chromium 在解析网页的 DOM 树的时候，每一个 DOM 元素都会计算出一个样式（因为 CSS 是零散分布的，浏览器还会有一些默认的 CSS）。这个样式会拥有一个 Font 和 FontDescription 元素。FontDescription 元素比较简单了，基本上就是 "font-size: 11 pt; font-family: sans-serif" 原样来的。Font 拥有两个内容，一个是 CssFontSelector, 一个是 FontFallbackList。

初始化 FontFallbackList 的时候会把 CssFontSelector 喂给它, 而 FontFallbackList 最重要的函数是：

	const FontData* FontFallbackList::GetFontData(const FontDescription& font_description)

主要是把 FontDescription 解析成真正的字体：先查找 FontCache, 缓存分为网页字体的缓存和系统字体的缓存（这就是 CssFontSelector 的作用，它是二合一的选择器，而 FontSelector 只是系统字体的选择器）; 如果缓存没有，针对 `font: 14pt 'Open Sans' sans-serif` 这样的 FontDescription 中的每个 font-family, 调用 ReportFontLookupByUniqueOrFamilyName。以上每一个 family name 都没有找到，会有一步 GetLastResortFallbackFont。那两个名字特别长的函数中间会转好几步，最后归根到底在 [font_fallback_linux.cc](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/font_fallback_linux.cc)。

我 2020 年第一遍看的时候是卡在 CssFontSelector 喂给 FontFallbackList 做初始化的时候，发现那个构造函数没有 Body。后来反应过来，没有 Body 无所谓啊，因为前面讲的一直是怎么从 css 到 Font/FontDescription，Font 又怎么到 FontFallbackList, 给 FontFallbackList 喂 FontDescription 就实质 populate FontFallbackList 了。

官方说明到这里就非常突兀地开始介绍 Text Shaping 了，就是把一段文字，按照 Unicode 做切分（Segmentation），然后用 Primary Font 去覆盖尽可能多的 Unicode, 剩下的用 Secondary Font 覆盖，一直到最后所有的文字都有字体显示。这里面最容易混淆的有一句话就是说，`如果 css 中列明的字体都没有覆盖这个 Unicode, 会调用 FallbackFontForCharacter 来显示这个 Unicode`, 即字体查询这时不再受限制于 css 中列明的字体，系统字体也会参与进来，即使没有明确的写在 css 里。因为有一个 `FallbackFontForCharacter`，所以就有人理解为 Chromium 做 Fallback 是单个字蹦的，这是不对的。

这时应该参照另一个官方文档：[RenderText and Chrome UI text drawing](https://www.chromium.org/developers/design-documents/rendertext/)。

里面说了 Linux 下的 Chromium 是用 harfbuzz 做渲染的，所以最重要的函数都在 [render_text_harfbuzz.cc](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/render_text_harfbuzz.cc) 里面，其中 `RenderTextHarfbuzz::ShapeRuns` 是最重要的：

它会先用 primary configured fonts from font_list() 的字体 Shape 一轮。剩下没匹配上的 Unicode 用 primary font 的 FallbackFont 再 Shape 一轮（比如你配置的字体是 Noto Sans CJK SC 跑了首轮, 第二轮就用它的 fallback font）。第三轮是用 fallback_font_list 跑，fallback_font_list 是用 GetFallbackFonts(primary_font) 生成的，最终也是回到 [font_fallback_linux.cc](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/font_fallback_linux.cc)。

这里面就是我非常熟悉的 fontconfig 了，核心代码：

    FallbackFontList fallback_fonts;
    FcPattern* pattern = FcPatternCreate();
    FcPatternAddString(pattern, FC_FAMILY,
                     reinterpret_cast<const FcChar8*>(font_family.c_str()));

    FcConfig* config = GetGlobalFontConfig();
    
    if (FcConfigSubstitute(config, pattern, FcMatchPattern) == FcTrue) {
      FcDefaultSubstitute(pattern);
      FcResult result;
      FcFontSet* fonts = FcFontSort(config, pattern, FcTrue, nullptr, &result);
    }
 
核心是 FcFontSort->FcFontSetSort。本质是基于 score 决定排序，score 主要是由 FcCompare->FcCompareValueList 生成。

对我来说其实 FcFontSort 不重要，我只需要知道 Chromium 使用了 FcConfigSubstitute 即可，它还是尊重 fontconfig 的。

另外第二轮用 GetFallbackFont(Primary font) 跑的时候，确实只会返回一个字体，但那也是先通过 FcFontSort 取结果，然后比较 charset 找到一个 coverage 最高的。

至于最早的那个 `FallbackFontForCharacter`，只见于 Blink 的 FontCache，而且这个功能只有 services/font 在用。根据 services 的定义，这不是 Chromium 的核心功能，更像是插件开发，所以就没继续研究。

而且我觉得，Text Shaping 先为每一个 Dom 元素生成 FontFallbackList，Skia 渲染的时候再 GetFontData, 到了这步的时候，有一两个遗漏的逐个 character 的取，好像也没什么毛病。就是不知道是不是这个顺序，后续有时间再研究。

### 其他

我还发现了 Chromium 是如何处理文本中的 Emoji 的，它会将与 Emoji 相关的 Unicode 作为单独的一次渲染处理。然后 FontCache 查找的时候，发现 Emoji 会优先在 und-zsye 这个 locale 里找字体。

它的 emoji 判断代码感觉也比较巧妙：

    bool IsEmojiRelatedCodepoint(UChar32 codepoint) {
      return u_hasBinaryProperty(codepoint, UCHAR_EMOJI) ||
         u_hasBinaryProperty(codepoint, UCHAR_EMOJI_PRESENTATION) ||
         u_hasBinaryProperty(codepoint, UCHAR_REGIONAL_INDICATOR);
    }

现在我的 fonts-config-ng 是先找 emoij 字体，再扫描它有什么 charset，如果用这个方法，应该可以减少一轮扫描。






