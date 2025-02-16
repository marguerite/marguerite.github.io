---
title: "深入理解 Linux Fontconfig 之五：构建 Chromium 字体调试环境"
date: 2025-02-12T00:00:00+08:00
draft: false
---

上一篇文章我们提到了 content_shell 可以用来调试 blink 引擎，今天我们就来编译一份 content_shell 并魔改它用来调试网页字体查询。

## 编译 content_shell

主要参考了 [Checking out and building Chromium on Linux](https://chromium.googlesource.com/chromium/src/+/main/docs/linux/build_instructions.md)

Chromium 的源代码是使用它自己的 depot_tools 编译的，我们先下载一份，比如解压到了 ~/Dev/depot_tools

然后这份 depot_tools 我们需要修改一个地方，添加代理，来应对网络环境，编辑 cipd 的这个地方：

    if hash curl 2> /dev/null ; then 
      curl -x socks5h://localhost:65533 "${URL}" -s --show-error -f --retry 3 --retry-delay 5 -A "${USER_AGENT}" -L -o "${CIPD_CLIENT_TMP}"

然后就可以使用 fetch 去获取代码了。

具体编译我使用了以下脚本：

set_environment.sh:

    #!/bin/sh
    export PATH=$PATH:~/Dev/depot_tools
    export http_proxy=http://127.0.0.1:8128
    export https_proxy=http://127.0.0.1:8128
    export socks_proxy=socks5://127.0.0.1:65533
    export all_proxy=http://127.0.0.1:8128
    export NO_AUTH_BOTO_CONFIG=~/.boto
    
prepare_build.sh:

    #!/bin/sh
    . ./set_environment.sh
    pushd src
    gn gen --args="host_os=\"linux\" is_debug=true dcheck_always_on=false enable_nacl=false is_component_ffmpeg=true use_cups=false use_aura=true use_kerberos=false enable_vr=false optimize_webui=false enable_reading_list=false use_pulseaudio=false link_pulseaudio=false is_component_build=false use_gnome_keyring=false use_vaapi=false use_dbus=false enable_rust=false enable_hangout_services_extension=false enable_vulkan=false use_lld=true is_clang=true" out/Debug
    popd

其中 gn gen 的 args 部分我主要参考了 openSUSE 下 chromium 的编译设置。

build.sh:

    #!/bin/sh
    . ./set_environment.sh
    autoninja -C ./src/out/Debug $1
    
然后使用

    ./build.sh content_shell
    
编译 content_shell。

## 启用 Logging

编译好的 content_shell 在 ./src/out/Debug。

我们先创建一个测试用的 html

    <html>
        <head/>
        <body>
            <h1 style="font-family: 'Segoe UI','Roboto',sans-serif; font-size: 20pt;">你好😃，我能吃玻璃而不伤身体</h1>
        </body>
    </html>
    
然后 content_shell 要启用 logging（因为 chromium 就自己的 Log 机制，所以纯 cpp 的 printf 没输出）：

content_shell_wrapper.sh:

    #!/bin/sh
    LC_NUMERIC="C" ./src/out/Debug/content_shell --no-sandbox --single-process --enable-logging=stderr --vmodule="\*/fonts/\*=4, \*/css/\*=0" --v=-3 > log.txt 2>&1 $1

要启用 `--single-process` 必须把 `LC_NUMERIC` 设为默认的 `C`，openSUSE 下它是空的。不然 `third_party/blink/renderer/platform/wtf/text/wtf_string.ccwtf_string.cc` 中的一个 `DCHECK_EQ(strcmp(setlocale(LC_NUMERIC, NULL), "C"), 0);` 会失败，造成 content_shell 崩溃。

--vmodule  的意思是只要路径中含有 fonts 和 css 的日志，--v=-3 的意思是压制其他日志。跑一下：

    ./content_shell_wrapper.sh test.html
    [6197:1:0212/100126.380015:VERBOSE4:shape_result_bloberizer.cc(513)] FillGlyphsNG fast path

我们只发现了一条，因为 Chromium 的开发者针对字体没有过多的 DEBUG 输出，所以我们要自己加...加的方法可以通过 shape_result_bloberizer.cc 看到：

先添加头文件：

    #include <base/logging.h>
    
然后在需要输出日志的地方：

    DVLOG(4) << "   SetText from: " << from << " to: " << to;
    
## 添加字体调试的日志输出

我们在 third_party/blink/renderer/platform/fonts/shaping 的 harfbuzz_shaper.cc 里加一条日志：

    // our dlog test
    const FontFamily* curr_family = &font_description.Family();
    DVLOG(0) << "fontdescription in HarfBuzzShaper::ShapeSegment";
    for (; curr_family; curr_family = curr_family->Next()) {
        DVLOG(0) << " current font_family is " << curr_family->FamilyName();
    }

然后重新编译一下 content_shell 再看。

注意：content_shell_wrapper.sh 里面的 loglevel 要从 4 改为 0。loglevel 是输出这个 level 及以下的，所以数字越小，内容越少。另外，虽然只改动了一个文件，但重新 LINK content_shell 的时间非常长，所以真正使用的时候，最好一次性把需要 DVLOG 输出的地方都改好再重新编译 content_shell。

运行结果如下：

    [18085:18085:0212/111541.814978:INFO:harfbuzz_shaper.cc(754)] fontdescription in HarfBuzzShaper::ShapeSegment
    [18085:18085:0212/111541.815081:INFO:harfbuzz_shaper.cc(756)]  current font_family is "Segoe UI"
    [18085:18085:0212/111541.815120:INFO:harfbuzz_shaper.cc(756)]  current font_family is "Roboto"
    [18085:18085:0212/111541.815152:INFO:harfbuzz_shaper.cc(756)]  current font_family is "sans-serif"

以上只是简单示例，距离产出完整的字体调试输出还需要修改好多地方。完整版的补丁在 [marguerite's blink_font_stack_debug.patch](https://gist.github.com/marguerite/763fab09a7e0c072880bacdef837b220)，最终编译的 content_shell 有 2GB 大小，有兴趣的可以自己去编译下。我是基于23年6月的一个 commit 编译的，本来打算更新到最新代码来着，但发现我的代码树是 cherry pick 的，就已经有 50GB 了，想想还是算了。两年前的代码也不是不能用，反正是为了验证基础逻辑。

## 字体加载阶段

[原理篇](https://marguerite.su/posts/getting_to_know_fontconfig_part_four_how_chromium_do_font_fallback/)里面已经说了，blink 在解析 DOM 树时，会针对每个 dom 元素计算出一个样式（ComputedStyle），该样式拥有 FontDescription 和 Font Object。Font Object 是由 FontBuilder 生成的，拥有一个 CssFontSelector 和一个 FontFallbackList。具体到 Debug Log 里面是这样的：

    [INFO:font.cc(83)] [Font(font_description, font_selector)] Font constructor called with font_selector 0x7fff553008c0
    [INFO:font_builder.cc(559)] [FontBuilder::CreateFont] FontDescription{family: "Segoe UI, Roboto, sans-serif", computedsize: 26.6667}
    [INFO:font_builder.cc(534)] [FontBuilder::ComputeFontSelector] FontSelectorFromTreeScope
    [INFO:font_builder.cc(526)]  [FontBuilder::FontSelectorFromTreeScope] tree_scope outerHTML "<body>\n<h1 style=\"font-family: 'Segoe UI','Roboto',sans-serif; font-size:20pt;\">\u4F60\u597D\uD83D\uDE03\uFF0C\u6211\u80FD\u5403\u73BB\u7483\u800C\u4E0D\u4F24\u8EAB\u4F53</h1>\n\n\n</body>"
    [INFO:font_builder.cc(527)] [FontBuilder::FontSelectorFromTreeScope] tree_scope innerHTML "\n<h1 style=\"font-family: 'Segoe UI','Roboto',sans-serif; font-size:20pt;\">\u4F60\u597D\uD83D\uDE03\uFF0C\u6211\u80FD\u5403\u73BB\u7483\u800C\u4E0D\u4F24\u8EAB\u4F53</h1>\n\n\n"
    [INFO:font.cc(64)] running GetOrCreateFontFallbackList
    [INFO:font.cc(65)] calling GetFontFallbackMap(font_selector).Get(font_description)
    [INFO:font.cc(66)] [GetOrCreateFontFallbackList] font_description "Segoe UI, Roboto, sans-serif"
    [INFO:font_fallback_map.cc(31)] [FontFallbackMap::Get] running FontFallbacklist::Create(*this fontfallbackmap)

先有 DOM，css style resolver 去解析 DOM（ StyleResolver::ComputeValue），一方面计算 ComputedStyle, 一方面生成 StyleResolverState, 后者在 StyleResolver::ComputeFont 阶段就会触发 UpdateFont。然后 FontBuilder 就会 CreateFont, 在这个函数从 DOM 计算 CssFontSelector。

在建字体的时候由于传入了 FontSelector，会保证建出该字体的 Fallback List，也就是到 `[FontFallbackMap::Get] running FontFallbacklist::Create(*this fontfallbackmap)` 这步。

然后 css 引擎触发 FontCache::Get().GetFontData() 的时候（最常见的是 CssFontSelector.GetFontData），或者更加直接如 FontDataAt() 的时候，才会去尝试用真正的字体填充。

    [INFO:font_fallback_list.cc(292)] [FontFallbackList::FontDataAt] Running GetFontData, font_description"Segoe UI, Roboto, sans-serif", realized_font_index 0
    [INFO:font_fallback_list.cc(176)] [FontFallbackList::GetFontData] curr_family "Segoe UI"
    [INFO:font_fallback_list.cc(191)] [FontFallbackList::GetFontData] running font_selector_.GetFontData(), font_description"Segoe UI, Roboto, sans-serif", family_name"Segoe UI"
    [INFO:font_fallback_list.cc(196)] [FontFallbackList::GetFontData] running FontCache::Get().GetFontData(), font_description"Segoe UI, Roboto, sans-serif", family_name"Segoe UI"
    [INFO:font_cache.cc(88)] running FontCache::Get()
    [INFO:font_cache.cc(244)] [FontCache::GetFontData] family "Segoe UI"
    [INFO:font_cache.cc(168)] running FontCache::GetFontPlatformData() 
    [INFO:font_cache.cc(169)] [FontCache::GetFontPlatformData] font_description"Segoe UI, Roboto, sans-serif"
    [INFO:font_cache.cc(170)] [FontCache::GetFontPlatformData] creation_params creation_type kCreateFontByFamily
    [INFO:font_cache.cc(172)] [FontCache::GetFontPlatformData] creation_params family "Segoe UI"
    [INFO:font_cache.cc(178)] [FontCache::GetFontPlatformData] alternative_font_name kAllowAlternate
    [INFO:font_platform_data_cache.cc(75)] [FontPlatformDataCache::GetOrCreateFontPlatformData] FontCacheKey hash 2683763
    [INFO:font_platform_data_cache.cc(88)] [FontPlatformDataCache::GetOrCreateFontPlatformData] font-size 26.66
    [INFO:font_platform_data_cache.cc(89)] [FontPlatformDataCache::GetOrCreateFontPlatformData] rounded_size 2666
    [INFO:font_platform_data_cache.cc(196)] [FontPlatformDataCache::SizedFontPlatformDataSet::GetOrCreateFontPlatformData] running font_cache->CreateFontPlatformData
    [INFO:font_cache_skia.cc(298)] [FontCache::CreateFontPlatformData] CreateTypeface(font_description, creation_params, name)
    [INFO:font_cache_skia.cc(299)] [FontCache::CreateFontPlatformData] font_description "Segoe UI, Roboto, sans-serif"
    [INFO:font_cache_skia.cc(300)] [FontCache::CreateFontPlatformData] creation_params creation_type kCreateFontByFamily
    [INFO:font_cache_skia.cc(302)] [FontCache::CreateFontPlatformData] creation_params family "Segoe UI"
    [INFO:font_cache_skia.cc(260)] [FontCache::CreateTypeface] initializing new SkFontMgr::RefDefault()
    [INFO:font_cache_skia.cc(263)] [FontCache::CreateTypeface] font_manager->matchFamilyStyle
    [INFO:font_cache_skia.cc(265)] [FontCache::CreateTypeface] name.c_str() Segoe UI
    [INFO:font_cache_skia.cc(266)] [FontCache::CreateTypeface] font_description.SkiaFontStyle(width: 5, weight: 700, slant: 0)
    [INFO:font_cache_skia.cc(317)] [FontCache::CreateFontPlatformData] NULL typeface Segoe UI

建不成 Skia SkTypeface 的日志长这样，就是最后有一个 NULL 然后就返回空指针了。能建成的最后长这样：

    [INFO:font_cache_skia.cc(298)] [FontCache::CreateFontPlatformData] CreateTypeface(font_description, creation_params, name)
    [INFO:font_cache_skia.cc(299)] [FontCache::CreateFontPlatformData] font_description "Segoe UI, Roboto, sans-serif"
    [INFO:font_cache_skia.cc(300)] [FontCache::CreateFontPlatformData] creation_params creation_type kCreateFontByFamily
    [INFO:font_cache_skia.cc(302)] [FontCache::CreateFontPlatformData] creation_params family "Sans"
    [INFO:font_cache_skia.cc(260)] [FontCache::CreateTypeface] initializing new SkFontMgr::RefDefault()
    [INFO:font_cache_skia.cc(263)] [FontCache::CreateTypeface] font_manager->matchFamilyStyle
    [INFO:font_cache_skia.cc(265)] [FontCache::CreateTypeface] name.c_str() Sans
    [INFO:font_cache_skia.cc(266)] [FontCache::CreateTypeface] font_description.SkiaFontStyle(width: 5, weight: 700, slant: 0)
    [INFO:font_cache_skia.cc(341)] [FontCache::CreateFontPlatformData] std::make_unique<FontPlatformData> name Sans
    [INFO:font_cache_skia.cc(344)] [FontCache::CreateFontPlatformData] typeface getFamilyName() Noto Sans CJK SC
    [INFO:font_cache_skia.cc(345)] [FontCache::CreateFontPlatformData] font_size 26.66
    [INFO:font_cache_skia.cc(347)] [FontCache::CreateFontPlatformData] synthetic_bold 0
    [INFO:font_cache_skia.cc(349)] [FontCache::CreateFontPlatformData] synthetic_italic 0
    [INFO:font_cache_skia.cc(350)] [FontCache::CreateFontPlatformData] text_rendering "Auto"
    [INFO:font_cache_skia.cc(361)] [FontCache::CreateFontPlatformData] font_platform_data.FontFamilyName() "Noto Sans CJK SC"
    [INFO:font_fallback_list.cc(295)] [FontFallbackList::FontDataAt] Got result

这里面与 fontconfig 能够联系上的是 `SkFontMgr::RefDefault()`，这是 chromium 自己的[实现](https://source.chromium.org/chromium/chromium/src/+/main:skia/ext/font_utils.cc)：

      sk_sp<SkFontConfigInterface> fci(SkFontConfigInterface::RefGlobal());
      return fci ? SkFontMgr_New_FCI(std::move(fci)) : nullptr;
      
最终是到 [SkFontConfigInterface_direct.cpp](https://source.chromium.org/chromium/chromium/src/+/main:third_party/skia/src/ports/SkFontConfigInterface_direct.cpp)

## 字体匹配阶段

原理篇也说了，先将 text 按照 unicode 切分，然后按照每一段 unicode 相同的文本选择对应的字体。具体到 Debug Log 长这样：

    [INFO:harfbuzz_shaper.cc(754)] [HarfBuzzShaper::ShapeSegment] font_description: "Segoe UI, Roboto, sans-serif"
    [INFO:harfbuzz_shaper.cc(755)] [HarfBuzzShaper::ShapeSegment] font_fallback_priority: kText
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:harfbuzz_shaper.cc(790)] [HarfbuzzShaper::ShapeSegment] fallback_chars_hint: "你"
    [INFO:font_fallback_iterator.cc(127)] [FontFallbackIterator::Next] fallball_stage_: kFontGroupFonts
    [INFO:font_fallback_iterator.cc(197)] [FontFallbackIterator::Next] Running font_fallback_list_->FontDataAt
    [INFO:font_fallback_iterator.cc(198)] [FontFallbackIterator::Next] font_description_ "Segoe UI, Roboto, sans-serif"
    [INFO:font_fallback_iterator.cc(199)] [FontFallbackIterator::Next] current_font_data_index_(realized_font_index) 0
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:font_fallback_iterator.cc(216)] [FontFallbackIterator::Next] font_data is not segmented 
    [INFO:harfbuzz_shaper.cc(754)] [HarfBuzzShaper::ShapeSegment] font_description: "Segoe UI, Roboto, sans-serif"
    [INFO:harfbuzz_shaper.cc(755)] [HarfBuzzShaper::ShapeSegment] font_fallback_priority: kEmojiEmoji
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:harfbuzz_shaper.cc(790)] [HarfbuzzShaper::ShapeSegment] fallback_chars_hint: "😃"
    [INFO:font_fallback_iterator.cc(127)] [FontFallbackIterator::Next] fallball_stage_: kFontGroupFonts
    [INFO:font_fallback_iterator.cc(197)] [FontFallbackIterator::Next] Running font_fallback_list_->FontDataAt
    [INFO:font_fallback_iterator.cc(198)] [FontFallbackIterator::Next] font_description_ "Segoe UI, Roboto, sans-serif"
    [INFO:font_fallback_iterator.cc(199)] [FontFallbackIterator::Next] current_font_data_index_(realized_font_index) 0
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:font_fallback_iterator.cc(216)] [FontFallbackIterator::Next] font_data is not segmented 
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:harfbuzz_shaper.cc(790)] [HarfbuzzShaper::ShapeSegment] fallback_chars_hint: "😃"
    [INFO:font_fallback_iterator.cc(127)] [FontFallbackIterator::Next] fallball_stage_: kFontGroupFonts
    [INFO:font_fallback_iterator.cc(197)] [FontFallbackIterator::Next] Running font_fallback_list_->FontDataAt
    [INFO:font_fallback_iterator.cc(198)] [FontFallbackIterator::Next] font_description_ "Segoe UI, Roboto, sans-serif"
    [INFO:font_fallback_iterator.cc(199)] [FontFallbackIterator::Next] current_font_data_index_(realized_font_index) 1
    [INFO:font_fallback_iterator.cc(127)] [FontFallbackIterator::Next] fallball_stage_: kFallbackPriorityFonts
    [INFO:font_fallback_iterator.cc(136)] [FontFallbackIterator::Next] FallbackPriorityFont
    [INFO:font_fallback_iterator.cc(270)] [FontFallbackIterator::FallbackPriorityFont] FontCache::Get().FallbackFontForCharacter
    [INFO:font_fallback_iterator.cc(271)] [FontFallbackIterator::FallbackPriorityFont] font_description_ "Segoe UI, Roboto, sans-serif"
    [INFO:font_fallback_iterator.cc(272)] [FontFallbackIterator::FallbackPriorityFont] hint 😃
    [NFO:font_fallback_iterator.cc(273)] [FontFallbackIterator::FallbackPriorityFont] fallback_priority_ kEmojiEmoji
    [INFO:font_fallback_iterator.cc(274)] [FontFallbackIterator::FallbackPriorityFont] font_fallback_list_->PrimarySimpleFontData(font_description_)"Noto Sans CJK SC"
    [INFO:font_cache.cc(88)] running FontCache::Get()
    [INFO:font_cache.cc(168)] running FontCache::GetFontPlatformData()
    [INFO:font_cache.cc(169)] [FontCache::GetFontPlatformData] font_description"Segoe UI, Roboto, sans-serif"
    [INFO:font_cache.cc(170)] [FontCache::GetFontPlatformData] creation_params creation_type kCreateFontByFciIdAndTtcIndex
    [INFO:font_cache.cc(174)] [FontCache::GetFontPlatformData] creation_params filename /usr/share/fonts/truetype/NotoColorEmoji.ttf
    [INFO:font_cache.cc(175)] [FontCache::GetFontPlatformData] creation_params FontconfigInterfaceId 0
    [INFO:font_cache.cc(176)] [FontCache::GetFontPlatformData] creation_params TtcIndex 0
    [INFO:font_cache.cc(178)] [FontCache::GetFontPlatformData] alternative_font_name kAllowAlternate
    [INFO:font_platform_data_cache.cc(75)] [FontPlatformDataCache::GetOrCreateFontPlatformData] FontCacheKey hash 13982713
    [INFO:font_platform_data_cache.cc(88)] [FontPlatformDataCache::GetOrCreateFontPlatformData] font-size 26.66
    [INFO:font_platform_data_cache.cc(89)] [FontPlatformDataCache::GetOrCreateFontPlatformData] rounded_size 2666
    [INFO:font_platform_data_cache.cc(196)] [FontPlatformDataCache::SizedFontPlatformDataSet::GetOrCreateFontPlatformData] running font_cache->CreateFontPlatformData
    [INFO:font_cache_skia.cc(298)] [FontCache::CreateFontPlatformData] CreateTypeface(font_description, creation_params, name)
    [INFO:font_cache_skia.cc(299)] [FontCache::CreateFontPlatformData] font_description "Segoe UI, Roboto, sans-serif"
    [INFO:font_cache_skia.cc(300)] [FontCache::CreateFontPlatformData] creation_params creation_type kCreateFontByFciIdAndTtcIndex
    [INFO:font_cache_skia.cc(304)] [FontCache::CreateFontPlatformData] creation_params filename /usr/share/fonts/truetype/NotoColorEmoji.ttf
    [INFO:font_cache_skia.cc(305)] [FontCache::CreateFontPlatformData] creation_params FontconfigInterfaceId 0
    [INFO:font_cache_skia.cc(306)] [FontCache::CreateFontPlatformData] creation_params TtcIndex 0
    [INFO:font_cache_skia.cc(230)] [FontCache::CreateTypeface] creation_params.CreationType() is kCreateFontByFciIdAndTtcIndex
    [INFO:font_cache_skia.cc(341)] [FontCache::CreateFontPlatformData] std::make_unique<FontPlatformData> name 
    [INFO:font_cache_skia.cc(344)] [FontCache::CreateFontPlatformData] typeface getFamilyName() Noto Color Emoji
    [INFO:font_cache_skia.cc(345)] [FontCache::CreateFontPlatformData] font_size 26.66
    [INFO:font_cache_skia.cc(347)] [FontCache::CreateFontPlatformData] synthetic_bold 0
    [INFO:font_cache_skia.cc(349)] [FontCache::CreateFontPlatformData] synthetic_italic 0
    [INFO:font_cache_skia.cc(350)] [FontCache::CreateFontPlatformData] text_rendering "Auto"
    [INFO:font_cache_skia.cc(361)] [FontCache::CreateFontPlatformData] font_platform_data.FontFamilyName() "Noto Color Emoji"
    [INFO:harfbuzz_shaper.cc(754)] [HarfBuzzShaper::ShapeSegment] font_description: "Segoe UI, Roboto, sans-serif"
    [INFO:harfbuzz_shaper.cc(755)] [HarfBuzzShaper::ShapeSegment] font_fallback_priority: kText
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:font_fallback_iterator.cc(113)] [FontFallbackIterator::NeedsHintList] FontDataAt
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:harfbuzz_shaper.cc(790)] [HarfbuzzShaper::ShapeSegment] fallback_chars_hint: "，我"
    [INFO:font_fallback_iterator.cc(127)] [FontFallbackIterator::Next] fallball_stage_: kFontGroupFonts
    [INFO:font_fallback_iterator.cc(197)] [FontFallbackIterator::Next] Running font_fallback_list_->FontDataAt
    [INFO:font_fallback_iterator.cc(198)] [FontFallbackIterator::Next] font_description_ "Segoe UI, Roboto, sans-serif"
    [INFO:font_fallback_iterator.cc(199)] [FontFallbackIterator::Next] current_font_data_index_(realized_font_index) 0
    [INFO:font_fallback_list.cc(275)] [FontFallbackList::FontDataAt] This fallback font is already in our list: 
    [INFO:font_fallback_list.cc(276)] [FontFallbackList::FontDataAt] realized_font_index of font_list_ size (0/1)
    [INFO:font_fallback_list.cc(277)] [FontFallbackList::FontDataAt] the result FontData: "Noto Sans CJK SC"
    [INFO:font_fallback_iterator.cc(216)] [FontFallbackIterator::Next] font_data is not segmented 

文本是“你好😊，我能吃玻璃而不伤身体”，第一遍确定使用 Noto Sans CJK SC，一直在遇到颜文字前都不需要提示词，遇到颜文字后又调整字体又调整 fallback_priority。

## 发现的问题

## 日志里好多 Times New Roman 哪里来的？

如果是 Sans / Arial 我都能够理解，因为在代码里看见过，`GetLastResortFont` 的最后的两步是找它们。但是 Times New Roman 是怎么出来的？找了一圈，发现因为它是[默认字体](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/app/resources/locale_settings_linux.grd)，Linux 下 Chromium 的默认字体如下：

    standard_font_family: Times New Roman
    fixed_font_family: Monospace
    serif_font_family: Times New Roman
    sans_serif_font_family: Arial
    ntp_font_family: Roboto
    cursive_font_family: Comic Sans MS
    fantasy_font_family: Impact
    math_font_family: Latin Modern Math 
    
## sans-serif 的 Fallback 不好使，sans 好使？

我们前面已经说过 Skia 了，如果喂给它  sans-serif 应该会直接有 Noto Sans CJK SC 回来，为什么 NULL Typeface 呢？

两个原因：

首先是在 css 里面喂会被替换成 Arial：

    [INFO:css_font_selector.cc(279)] [CSSFontSelector::GetFontData] family_name: "sans-serif"
    [INFO:css_font_selector.cc(280)] [CSSFontSelector::GetFontData] settings_family_name: "Arial"
    [INFO:css_font_selector.cc(281)] [CSSFontSelector::GetFontData] generic family: "SansSerif"
    
其次是直接喂会变成这样：

    [INFO:font_cache_skia.cc(298)] [FontCache::CreateFontPlatformData] CreateTypeface(font_description, creation_params, name)
    [INFO:font_cache_skia.cc(299)] [FontCache::CreateFontPlatformData] font_description "Segoe UI, Roboto, sans-serif"
    [INFO:font_cache_skia.cc(300)] [FontCache::CreateFontPlatformData] creation_params creation_type kCreateFontByFamily
    [INFO:font_cache_skia.cc(302)] [FontCache::CreateFontPlatformData] creation_params family "sans-serif"
    [INFO:font_cache_skia.cc(260)] [FontCache::CreateTypeface] initializing new SkFontMgr::RefDefault()
    [INFO:font_cache_skia.cc(263)] [FontCache::CreateTypeface] font_manager->matchFamilyStyle
    [INFO:font_cache_skia.cc(265)] [FontCache::CreateTypeface] name.c_str() sans-serif
    [INFO:font_cache_skia.cc(266)] [FontCache::CreateTypeface] font_description.SkiaFontStyle(width: 5, weight: 700, slant: 0)
    [INFO:font_cache_skia.cc(317)] [FontCache::CreateFontPlatformData] NULL typeface sans-serif
    
会去要 Normal width, Bold weight, Upright slant 的 sans-serif。然后就不知道为什么网页里面没写加粗，会去要加粗的 sans-serif 了，另外，即使要粗体应该也不会一个没有，也许我应该写个 skia demo debug 一下。

## 制作 skia debug 工具

单独编译 skia 不太可取，因为我的 chromium 代码树下已经有 skia 了。于是就有了以下取巧的方案：

在 `src/skia/BUILD.gn` 里添加这样的代码：

    group("fuzzers") {
        deps = [ "//skia/tools/fuzzers" ]
    }

下面加上：

    group("skia_debug_tool") {
        deps = [ "//skia/skia_debug_tool" ]
    }

然后建立一个 `src/skia/skia_debug_tool` 文件夹，里面内容如下：

src/skia/skia_debug_tool/BUILD.gn:

    executable("skia_debug_tool") {
        sources = [
            "skia_debug_tool.cc",
            "fontmgr_default_linux.cc"
        ]

        deps = [ "//skia" ]
    }
    
src/skia/skia_debug_tool/skia_debug_tool.cc：

    #include "fontmgr_default_linux.h"
    #include <stdio.h>
    #include "../../third_party/skia/include/core/SkFontStyle.h"

    int main(int argc, char* argv[]) {
        auto font_mgr = skia_debug_tool::CreateDefaultSkFontMgr();
        printf("font_mgr created: ok\n");
        const char* family_name = "sans-serif";
        printf("family name created: ok\n");
        auto style = SkFontStyle(5, 700, SkFontStyle::kUpright_Slant);
        auto typeface = font_mgr->matchFamilyStyle(family_name, style);
        if (typeface) {
            printf("typeface is not null\n");
        } else {
            printf("typeface is null\n");
        }
        return 0;
    }
    
src/skia/skia_debug_tool/fontmgr_default_linux.cc:

    #include "../../third_party/skia/include/core/SkFontMgr.h"
    #include "../../third_party/skia/include/ports/SkFontConfigInterface.h"
    #include "../../third_party/skia/include/ports/SkFontMgr_FontConfigInterface.h"

    namespace skia_debug_tool {

        SK_API sk_sp<SkFontMgr> CreateDefaultSkFontMgr() {
            sk_sp<SkFontConfigInterface> fci(SkFontConfigInterface::RefGlobal());
            return fci ? SkFontMgr_New_FCI(std::move(fci)) : nullptr;
        }

    }  // namespace skia_debug_tool

src/skia/skia_debug_tool/fontmgr_default_linux.h:

    #include "../../third_party/skia/include/core/SkFontMgr.h"
    #include "../../third_party/skia/include/ports/SkFontConfigInterface.h"
    #include "../../third_party/skia/include/ports/SkFontMgr_FontConfigInterface.h"

    namespace skia_debug_tool {

        SK_API sk_sp<SkFontMgr> CreateDefaultSkFontMgr();

    }  // namespace skia_debug_tool

后面两个的代码是从 src/third_party/skia/src/ports/fontmgr_default_linux.cc 抄来改的，就是为了创建跟 blink 一模一样的 fontmgr 来用。而 skia_debug_tool.cc 的代码是从 CreateTypeface 抄来的，就是想验证一下单独跑会不会是一个结果。

重新 `prepare_build.sh`，然后 `build.sh skia_debug_tool`，最后运行：

    ./src/out/Debug/skia_debug_tool
    
得到：

    font_mgr created: ok
    family name created: ok
    typeface is null
    
与 blink 返回的结果是一样的。我们要一下 normal weight（即把 700 改为 400）看看：

    font_mgr created: ok
    family name created: ok
    typeface is null
    
结果是一样的。不是因为字体 weight 的问题导致的 skia 取不到 sans-serif 的 typeface。再调试这个问题之前，我们先来看一下 sans/Sans/SansSerif 的：sans/Sans 能够成功，SansSerif/sansserif/Sans-Serif 依然为空。

## 调试 Skia 的 fontconfig

只好继续添加调试代码，`src/third_party/skia/src/ports/SkFontConfigInterface_direct.cpp` 的调试代码如下：

    bool SkFontConfigInterfaceDirect::matchFamilyName(const char familyName[],
                                                  SkFontStyle style,
                                                  FontIdentity* outIdentity,
                                                  SkString* outFamilyName,
                                                  SkFontStyle* outStyle) {
        SkString familyStr(familyName ? familyName : "");

        printf("running SkFontConfigInterfaceDirect::matchFamilyName with family name: ");
        printf("%s", familyName);
        printf("\n");
        // printf 都是我加的调试代码，没有调试代码的部分略
        ...

        FcPatternAddBool(pattern, FC_SCALABLE, FcTrue);

        printf("Running FcConfigSubstitute:\n");
        FcPatternPrint(pattern);
        FcConfigSubstitute(fc, pattern, FcMatchPattern);
        printf("After running FcDefaultSubstitute:\n");
        FcDefaultSubstitute(pattern);
        FcPatternPrint(pattern);

        ...
        
        printf("post_config_family: %s\n", post_config_family);

        FcResult result;
        FcFontSet* font_set = FcFontSort(fc, pattern, 0, nullptr, &result);
        if (!font_set) {
            printf("FcFontSort didn't get a FontSet\n");
            FcPatternDestroy(pattern);
            return false;
        }

        FcPattern* match = this->MatchFont(font_set, post_config_family, familyStr);
        if (!match) {
            printf("post_config_family: %s\n", post_config_family);
            printf("familyStr: %s\n", familyStr.c_str());
            printf("MatchFont didn't get a match!\n");
            FcPatternDestroy(pattern);
            FcFontSetDestroy(font_set);
            return false;
        }

        FcPatternDestroy(pattern);

        // From here out we just extract our results from 'match'
        printf("Got a match!\n");
        post_config_family = get_string(match, FC_FAMILY);
        if (!post_config_family) {
            printf("Can't get a post_config_family!\n");
            FcFontSetDestroy(font_set);
            return false;
        }
        printf("Got post_config_family: %s\n", post_config_family);

之后输出如下：

    font_mgr created: ok
    family name created: ok
    running SkFontConfigInterfaceDirect::matchFamilyName with family name: sans-serif
    Running FcConfigSubstitute:
    Pattern has 5 elts (size 16)
        family: "sans-serif"(s)
        slant: 0(i)(s)
        weight: 0(i)(s)
        width: 200(i)(s)
        scalable: True(s)
    After running FcDefaultSubstitute:
    Pattern has 36 elts (size 48)
        family: "Noto Sans"(w) "Noto Sans SC"(w) "Noto Sans HK"(w) "Noto Sans TC"(w) "Noto Sans JP"(w) "Noto Sans KR"(w) "Noto Sans CJK SC"(w) "Arial"(w) "Albany AMT"(w) "Verdana"(w) "Roboto"(w) "Noto Kufi Arabic"(w) "Noto Naskh Arabic"(w) "Noto Sans"(w) "Noto Sans Armenian"(w) "Noto Sans Avestan"(w) "Noto Sans Balinese"(w) "Noto Sans Bamum"(w) "Noto Sans Batak"(w) "Noto Sans Bengali"(w) "Noto Sans Brahmi"(w) "Noto Sans Buginese"(w) "Noto Sans Buhid"(w) "Noto Sans Canadian Aboriginal"(w) "Noto Sans Carian"(w) "Noto Sans Cherokee"(w) "Noto Sans Coptic"(w) "Noto Sans Cypriot"(w) "Noto Sans Deseret"(w) "Noto Sans Devanagari"(w) "Noto Sans Egyptian Hieroglyphs"(w) "Noto Sans Ethiopic"(w) "Noto Sans Georgian"(w) "Noto Sans Glagolitic"(w) "Noto Sans Gothic"(w) "Noto Sans Gujarati"(w) "Noto Sans Gurmukhi"(w) "Noto Sans Hanunoo"(w) "Noto Sans Hebrew"(w) "Noto Sans Imperial Aramaic"(w) "Noto Sans Inscriptional Pahlavi"(w) "Noto Sans Inscriptional Parthian"(w) "Noto Sans JP"(w) "Noto Sans Javanese"(w) "Noto Sans Kaithi"(w) "Noto Sans Kannada"(w) "Noto Sans Kayah Li"(w) "Noto Sans Kharoshthi"(w) "Noto Sans KR"(w) "Noto Sans Lao"(w) "Noto Sans Lepcha"(w) "Noto Sans Limbu"(w) "Noto Sans Linear B"(w) "Noto Sans Lisu"(w) "Noto Sans Lycian"(w) "Noto Sans Lydian"(w) "Noto Sans Malayalam"(w) "Noto Sans Mandaic"(w) "Noto Sans Meetei Mayek"(w) "Noto Sans Mongolian"(w) "Noto Sans Myanmar"(w) "Noto Sans New Tai Lue"(w) "Noto Sans NKo"(w) "Noto Sans Ogham"(w) "Noto Sans Old Italic"(w) "Noto Sans Old Persian"(w) "Noto Sans Old South Arabian"(w) "Noto Sans Old Turkic"(w) "Noto Sans Ol Chiki"(w) "Noto Sans Osmanya"(w) "Noto Sans Phags-pa"(w) "Noto Sans Phoenician"(w) "Noto Sans Rejang"(w) "Noto Sans Runic"(w) "Noto Sans Samaritan"(w) "Noto Sans Saurashtra"(w) "Noto Sans Shavian"(w) "Noto Sans Sinhala"(w) "Noto Sans Sumero-Akkadian Cuneiform"(w) "Noto Sans Sundanese"(w) "Noto Sans Syloti Nagri"(w) "Noto Sans Symbols"(w) "Noto Sans Syriac Eastern"(w) "Noto Sans Syriac Estrangela"(w) "Noto Sans Syriac Western"(w) "Noto Sans SC"(w) "Noto Sans Tagalog"(w) "Noto Sans Tagbanwa"(w) "Noto Sans Tai Le"(w) "Noto Sans Tai Tham"(w) "Noto Sans Tai Viet"(w) "Noto Sans Tamil"(w) "Noto Sans Telugu"(w) "Noto Sans Thai"(w) "Noto Sans Tifinagh"(w) "Noto Sans TC"(w) "Noto Sans Ugaritic"(w) "Noto Sans Vai"(w) "Noto Sans Yi"(w) "Liberation Sans"(w) "Droid Sans"(w) "Arimo"(w) "Cantarell"(w) "SUSE Sans"(w) "Bitstream Vera Sans"(w) "Nimbus Sans L"(w) "Luxi Sans"(w) "Mukti Narrow"(w) "KacstBook"(w) "Nachlieli CLM"(w) "Helvetica"(w) "Khmer OS System"(w) "Lohit Punjabi"(w) "Lohit Oriya"(w) "Pothana2000"(w) "TSCu_Paranar"(w) "BPG Glaho"(w) "Terafik"(w) "FreeSans"(w) "Meiryo"(w) "MS PGothic"(w) "MS Gothic"(w) "HGPGothicB"(w) "HGGothicB"(w) "IPAPGothic"(w) "IPAGothic"(w) "IPAexGothic"(w) "VL PGothic"(w) "VL Gothic"(w) "Sazanami Gothic"(w) "Kochi Gothic"(w) "CMEXSong"(w) "FZSongTi"(w) "WenQuanYi Micro Hei"(w) "WenQuanYi WenQuanYi Bitmap Song"(w) "WenQuanYi Zen Hei"(w) "AR PL ShanHeiSun Uni"(w) "FZMingTiB"(w) "AR PL SungtiL GB"(w) "AR PL Mingti2L Big5"(w) "NanumGothic"(w) "UnDotum"(w) "Baekmuk Gulim"(w) "Baekmuk Dotum"(w) "Noto Sans"(w) "DejaVu Sans"(w) "Verdana"(w) "Arial"(w) "Albany AMT"(w) "Luxi Sans"(w) "Nimbus Sans L"(w) "Nimbus Sans"(w) "Helvetica"(w) "Lucida Sans Unicode"(w) "BPG Glaho International"(w) "Tahoma"(w) "Nachlieli"(w) "Lucida Sans Unicode"(w) "Yudit Unicode"(w) "Kerkis"(w) "ArmNet Helvetica"(w) "Artsounk"(w) "BPG UTF8 M"(w) "Waree"(w) "Loma"(w) "Garuda"(w) "Umpush"(w) "Saysettha Unicode"(w) "JG Lao Old Arial"(w) "GF Zemen Unicode"(w) "Pigiarniq"(w) "B Davat"(w) "B Compset"(w) "Kacst-Qr"(w) "Urdu Nastaliq Unicode"(w) "Raghindi"(w) "Mukti Narrow"(w) "malayalam"(w) "Sampige"(w) "padmaa"(w) "Hapax Berbère"(w) "MS Gothic"(w) "UmePlus P Gothic"(w) "Microsoft YaHei"(w) "Microsoft JhengHei"(w) "WenQuanYi Zen Hei"(w) "WenQuanYi Bitmap Song"(w) "AR PL ShanHeiSun Uni"(w) "AR PL New Sung"(w) "Hiragino Sans"(w) "PingFang SC"(w) "PingFang TC"(w) "PingFang HK"(w) "Hiragino Sans CNS"(w) "Hiragino Sans GB"(w) "MgOpen Modata"(w) "VL Gothic"(w) "IPAMonaGothic"(w) "IPAGothic"(w) "Sazanami Gothic"(w) "Kochi Gothic"(w) "AR PL KaitiM GB"(w) "AR PL KaitiM Big5"(w) "AR PL ShanHeiSun Uni"(w) "AR PL SungtiL GB"(w) "AR PL Mingti2L Big5"(w) "ＭＳ ゴシック"(w) "ZYSong18030"(w) "TSCu_Paranar"(w) "NanumGothic"(w) "UnDotum"(w) "Baekmuk Dotum"(w) "Baekmuk Gulim"(w) "Apple SD Gothic Neo"(w) "KacstQura"(w) "Lohit Bengali"(w) "Lohit Gujarati"(w) "Lohit Hindi"(w) "Lohit Marathi"(w) "Lohit Maithili"(w) "Lohit Kashmiri"(w) "Lohit Konkani"(w) "Lohit Nepali"(w) "Lohit Sindhi"(w) "Lohit Punjabi"(w) "Lohit Tamil"(w) "Meera"(w) "Lohit Malayalam"(w) "Lohit Kannada"(w) "Lohit Telugu"(w) "Lohit Oriya"(w) "LKLUG"(w) "FreeSans"(w) "Arial Unicode MS"(w) "Arial Unicode"(w) "Code2000"(w) "Code2001"(w) "sans-serif"(s) "Roya"(w) "Koodak"(w) "Terafik"(w)
        familylang: "zh-CN"(s) "en-us"(w)
        stylelang: "zh-CN"(s) "en-us"(w)
        fullnamelang: "zh-CN"(s) "en-us"(w)
        slant: 0(i)(s)
        weight: 0(i)(s)
        width: 200(i)(s)
        size: 12(f)(s)
        pixelsize: 12.5(f)(s)
        antialias: True(w)
        hintstyle: 1(i)(w)
        hinting: True(s)
        verticallayout: False(s)
        autohint: False(s)
        globaladvance: True(s)
        scalable: True(s)
        dpi: 75(f)(s)
        rgba: 1(i)(w) 5(i)(w)
        scale: 1(f)(s)
        lang: "zh-CN"(w)
        fontversion: 2147483647(i)(s)
        embeddedbitmap: True(s)
        decorative: False(s)
        lcdfilter: 1(i)(w) 1(i)(w)
        namelang: "zh-CN"(s)
        prgname: "skia_debug_tool"(s)
        symbol: False(s)
        variable: False(s)
        order: 0(i)(s)
        desktop: "KDE"(s)
        force_hintstyle: "hintslight"(w)
        force_autohint: False(w)
        force_bw: False(w)
        force_bw_monospace: False(w)
        search_metric_aliases: True(w)
        user_preference_list: True(w)

    post_config_family: Noto Sans
    post_config_family: Noto Sans
    familyStr: sans-serif
    MatchFont didn't get a match!
    typeface is null
    
 看到是这个 pattern match 有些问题。导致匹配到的是 Noto Sans。