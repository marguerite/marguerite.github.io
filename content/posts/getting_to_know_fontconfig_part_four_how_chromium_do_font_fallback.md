---
title: "深入理解 Linux Fontconfig 之四：Chromium 字体查询机制"
date: 2025-02-10T00:00:00+08:00
draft: false
---

要谈 Chromium 的字体查询机制，首先要分开 UI 字体和网页字体。UI 字体是渲染 Chromium 菜单栏中文本使用的，网页字体是渲染网页中的文本的。前者的代码主要在 ui/gfx 域下，后者是由 third_party/blink 渲染引擎处理。

## UI 字体的渲染

官方文档：[RenderText and Chrome UI text drawing](https://www.chromium.org/developers/design-documents/rendertext/)

Linux 下 Chromium 是用 harfbuzz 做文本渲染的，最重要的函数都在 [render_text_harfbuzz.cc](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/render_text_harfbuzz.cc) 里面，其中 `RenderTextHarfbuzz::ShapeRuns` 是最重要的：

它会先用 primary configured fonts from font_list() 的字体 Shape 一轮。剩下没匹配上的 Unicode 用 primary font 的 FallbackFont 再 Shape 一轮（比如你配置的字体是 Noto Sans CJK SC 跑了首轮, 第二轮就用它的 fallback font）。第三轮是用 fallback_font_list 跑，fallback_font_list 是用 GetFallbackFonts(primary_font) 生成的，最终是回到 [font_fallback_linux.cc](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/font_fallback_linux.cc)。

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

我们只需要知道 Chromium 使用了 FcConfigSubstitute 即可，UI 字体这部分它是尊重 fontconfig 的。

另外第二轮用 GetFallbackFont(Primary font) 跑的时候，确实只会返回一个字体，但也是先通过 FcFontSort 取结果，然后比较 charset 找到一个 coverage 最高的。

## 网页文本的渲染

官方文档：[Blink's Text Stack](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/platform/fonts/README.md)。

文档里提到了 `FallbackFontForCharacter`，这是很多人认为 Chromium 渲染是单字的源头。我也是想了解 Blink 引擎到底是怎么调用 fontconfig 的，才有了这篇文章。

Chromium 在解析网页的 DOM 树的时候，每一个 DOM 元素都会计算出一个样式（因为 CSS 是零散分布的，浏览器还会有一些默认的 CSS）。这个样式会拥有一个 Font 和 FontDescription 元素。FontDescription 元素比较简单了，基本上就是 "font-size: 11 pt; font-family: sans-serif" 原样来的。Font 拥有两个内容，一个是 CssFontSelector, 一个是 FontFallbackList。

FontFallbackList 最重要的函数是：

	const FontData* FontFallbackList::GetFontData(const FontDescription& font_description)

它会 loop font_description.Family() 返回的 FontFamily, 后者是一个 iter，其实就是把 css 中的 font-family 逐个返回。要系统字体的代码是 `FontCache::Get().GetFontData()`。如果 font_description 中给定的 family 都处理完都不合格，会先调用用户偏好设置的字体，最后是 `GetLastResortFallbackFont`。

FontCache 下的 `GetFontData(font_description, font_family)` 会调用 `FontPlatformDataCache::GetOrCreateFontPlatformData()`，最终到 [font_cache_skia.cc](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc) 中的 `FontCache::CreateFontPlatformData`->`FontPlatformData::CreateSkFont`。

Skia 在 Linux 上是用 fontconfig  的：[SkFontMgr_fontconfig.cpp](https://source.chromium.org/chromium/chromium/src/+/main:third_party/skia/src/ports/SkFontMgr_fontconfig.cpp)，它的 `onMatchFamily` 主要是这么搞的：

        SkAutoFcPattern pattern;
        FcPatternAddString(pattern, FC_FAMILY, (const FcChar8*)familyName);
        FcConfigSubstitute(fFC, pattern, FcMatchPattern);
        FcDefaultSubstitute(pattern);
        
根据我们在前几篇文章中的分析，这个结构处理 'sans-serif'，甚至我们切掉部分 charset 的 font 都是没问题的。

另外 Skia 还有一个 [SkFontConfigInterface_direct.cpp](https://source.chromium.org/chromium/chromium/src/+/main:third_party/skia/src/ports/SkFontConfigInterface_direct.cpp) 也可能更加重要。别的地方没有 fontconfig 了，Skia 只要用，就只能这么用。

FontFallbacklist  是针对每个 family 调用 FontCache::GetFontData，即便是最终到了 'sans-serif'，也有 Skia 兜底。

知道了 FontFallbackList 的数据是怎么来的，还需要知道它是怎么用的，才可以不盯着 FallbackForChar 不放。这需要我们再去研究一下 Blink 是怎么切文本的，如果它把每个文本都切成 char, 那 per char 的 fallback 就没有问题。

关键函数在 [harfbuzz_shaper.cc](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) 的  `HarfBuzzShaper::ShapeSegment`。它会建立一个 FontFallbackIterator 然后一直把 reshape_queue 跑干净。跑 Iter 的时候，会区分带 hint_list 的 runs 或 hint_list 为空的 runs。而是不是需要 hint_list 是由 FontFallbackIter 的 fallback_stage 处于哪一阶段决定的，segmented 和 kFontGroupFonts 需要提供提示字。

FontFallbackIter 在创建的时候会创建空白的 FontFallbackList，即 EnsureFontFallbacklist()。

FontFallbackIter 关键函数是  `FallbackPriorityFont`、`UniqueSystemFontForHintList`和`FontCache::GetLastResortFallbackFont`，顺序执行这三个函数。前两者都会调用 FallbackFontForChar，原因是 segmented 其实是一句话，但提示词只给了一个字，这个字就能决定这段话的 Unicode。中文是单独 segmented 的，如何 segment 在 [script_run_iterator.cc](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/fonts/script_run_iterator.cc)。全是中文的一段话里给出一个字其实已经够了，是为了效率的考虑。

`FontCache::GetLastResortFallbackFont` 其实就是使用 sans-serif，如果 sans-serif 没有用 sans, 再没有用 Arial, 再没有用 Courier New。Windows 还多维护了几个 Fallback 字体，在 [font_family_names.json5](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/fonts/font_family_names.json5)

`FallbackFontForCharacter` 兜兜转转，实现在 ui/gfx/font_fallback_linux.cc，其实就是封装了一个 `FcCharSetHasChar`。

## 其他

我还发现了 Chromium 是如何处理文本中的 Emoji 的，它在切词的时候发现文本中有 Emoji 会调整 FontFallbackPriority，从 kText 调整为 kEmojiText 这样，但是调整了后续却没做什么。然后 FontCache 查找的时候，发现 Emoji 会优先在 und-zsye 这个 locale 里找字体。

它的 emoji 判断代码感觉也比较巧妙：

    bool IsEmojiRelatedCodepoint(UChar32 codepoint) {
      return u_hasBinaryProperty(codepoint, UCHAR_EMOJI) ||
         u_hasBinaryProperty(codepoint, UCHAR_EMOJI_PRESENTATION) ||
         u_hasBinaryProperty(codepoint, UCHAR_REGIONAL_INDICATOR);
    }

现在我的 fonts-config-ng 是先找 emoij 字体，再扫描它有什么 charset，如果用这个方法，应该可以减少一轮扫描。

知道了原理，你可能也想去测试一些想法，我建议不要直接使用 Chromium 去测，可以考虑编译一下只包含了 Blink 引擎的 [Content Shell](https://www.chromium.org/blink/getting-started-with-blink-debugging/)。






