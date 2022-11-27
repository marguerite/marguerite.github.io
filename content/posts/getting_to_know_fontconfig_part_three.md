---
title: "深入理解 Linux fontconfig 第三部分"
date: 2022-11-26T22:02:04+08:00
draft: false
---
[上一篇](https://marguerite.su/posts/getting_to_know_fontconfig_part_two/)说到，我们 debug 了 `FcConfigSubstitute` 和 `FcConfigReference`，最后找到了 `FcFileScanFontConfig`，一番魔改后发现 `<match target="scan">` 虽然应用了，但是没有影响最终 FontSet 中 "Noto Scan CJK SC" 的 charset。

我们这篇要继续 debug：

    FcConfigSubstitute (config, font, FcMatchScan)
    
`FcFileScanFontConfig` 中的这段代码。

## 无踪无迹的指针

我们知道，上面的 font 是一个指针。要想前后输出有所变化，是一定要修改这个指针指向的数据的。也就是说，我们只需要去 `FcConfigSubsitute`里找修改指针指向数据的部分就可以了。所以我们只需要关注应用规则集时候的 `case FcRuleEdit` 里 `case FcOpAssign` 的部分就好。

通过 `if (value[object])` 这个判断我们找到了 test 跟 edit 的联系，原来是用 test 去找到要修改的部分：

    /* different 'kind' won't be the target of edit */
    if (!value[object] && kind == r->u.test->kind)
      value[object] = vl;
      
但是剩下的都是 `FcConfigAdd` 和 `FcConfigDel` 是修改 `config` 的，并不直接修改 Font。

正当我措手无策的时候，无意间看到了下面：

    case FcOpAssignReplace:
			/*
			 * Delete all of the values and insert
			 * the new set
			 */
			FcConfigPatternDel (p, r->u.edit->object, table);
			FcConfigPatternAdd (p, r->u.edit->object, l, FcTrue, table);
			/*
			 * Adjust a pointer into the value list as they no
			 * longer point to anything valid
			 */
			value[object] = NULL;
			break;
这家伙是修改 `pattern` 的啊！

## 语焉不详的文档

大家一定看到过文档中的这段：

      Mode                    With Match              Without Match
      ---------------------------------------------------------------------
      "assign"                Replace matching value  Replace all values
      "assign_replace"        Replace all values      Replace all values
      "prepend"               Insert before matching  Insert at head of list
      "prepend_first"         Insert at head of list  Insert at head of list
      "append"                Append after matching   Append at end of list
      "append_last"           Append at end of list   Append at end of list
      "delete"                Delete matching value   Delete all values
      "delete_all"            Delete all values       Delete all values
      
看起来 `assign` 和 `assign_replace` 没差。

但实际上差多了！

我把我的代码换成：

    <match target="scan">
      <test name="family"> 
        <string>Noto Sans CJK SC</string>
      </test>
      <edit name="charset" mode="assign_replace">
        <minus>
          <name>charset</name>
          <charset>
            <int>0x203c</int>
          </charset>
          <charset>
            <int>0x2122</int>
          </charset>
        </minus>  
      </edit> 
    </match>

再跑：

    Noto Sans CJK SC
    before match scan: Yes
    charset count: 44810
    after match scan: Yes
    charset count: 44809
    
成功去掉了‼字符！

于是：

**如果是 `<match target="scan">` 这种需要调整 `pattern` 的，用 `assign_replace`，如果是只调整 `config` 就可以的，比如 `<match target="pattern|font">`，用 `assign`**。

另外，在 fontconfig 里，万物皆 `pattern`，不要被 `<match target="pattern">` 迷惑了，实际上它代码中的 pattern：

    struct _FcPattern {
    int		    num;
    int		    size;
    intptr_t	    elts_offset;
    FcRef	    ref;
    };

就是指针的 struct。而对于 `FcPatternElt` 的注释也说了：

    /*
    * Pattern elts are stuck in a structure connected to the pattern,
    * so they get moved around when the pattern is resized. Hence, the
    * values field must be a pointer/offset instead of just an offset
    */

这是阅读源代码才得到的金科玉律啊！也就是说，别人说的，Firefox/Chromium 不支持调整 `charset`，是不对的，只是我们没会用 fontconfig，不要甩锅给 chromium 了！

## 为什么只去掉了一个字符

因为程序里写法错了，正确的写法是：

    <match target="scan">
      <test name="family"> 
        <string>Noto Sans CJK SC</string>
      </test>
      <edit name="charset" mode="assign_replace">
        <minus>
          <name>charset</name>
          <charset>
            <int>0x203c</int>
            <int>0x2122</int>
             <range>
              <int>0x1f250</int>
              <int>0x1f251</int>
            </range>
          </charset>
        </minus>
      </edit> 
    </match>

只有一个 `<charset></charset>` block。多余的会被丢弃。

我的问题解决了。下一步就是修改我的代码去了，顺便给大家一个去掉系统上已安装字体中 emoji 字符的 c 文件吧（写出来了放不出来，因为 fontconfig 关于 `FcCharSet*`相关方法的 Public 函数里没有 `FcCharSetIterStart` 和 `FcCharSetIterNext`，也没有 `FcNameUnparseCharSet`，没有办法 loop charset 里的 leaf 并打印。 我是魔改了 fontconfig 后写出来的...暂时只能自己用）

## 其他我在第一部分抛出问题的答案

首先，`FcConfigSubstitute` 删除 charset leaf，如果你是显式的用，就是在 `FcListPatternMatchAny`阶段，因为 `config` 里无论你是 `assign` 还是 `assign_replace` 都认的。如果你是隐式的用，那就是在 `FcFileScanFontConfig` 的时候，但是必须用 `assign_replace`。

其次，`FcConfigSubstitue` 只会做 `<match target="pattern">`，如果你的第一个参数也就是 `config` 传 0 会做 `<match target="scan">`。这是我验证过的。而 `<match target="font">` 只会在一个地方做：`FcFontRenderPrepare`（可以搜 `FcMatchFont` 查到）。另外 Chromium 的 ` GetFontRenderParamsFromFcPattern` 是没有调用 `FcFontRenderPrepare` 的，它只会取字体默认的 aa，autohint, embedded_bitmap, hinting 和 rgba。所以你们批评它也没错。

第三，双猫批评的 chromium 只返回一个 Fallback Font 的问题，确实存在，大概是在 [ui/gfc/render_text_harfbuzz.cc#RenderTextHarfBuzz::ShapeRuns](https://github.com/chromium/chromium/blob/main/ui/gfx/render_text_harfbuzz.cc)：

    const Font& primary_font = font_list().GetPrimaryFont();

    // Find fallback fonts for the remaining runs using a worklist algorithm. Try
    // to shape the first run by using GetFallbackFont(...) and then try shaping
    // other runs with the same font. If the first font can't be shaped, remove it
    // and continue with the remaining runs until the worklist is empty. The
    // fallback font returned by GetFallbackFont(...) depends on the text of the
    // run and the results may differ between runs.
    std::vector<internal::TextRunHarfBuzz*> remaining_unshaped_runs;
    while (!runs.empty()) {
    Font fallback_font(primary_font);
    bool fallback_found;
    internal::TextRunHarfBuzz* current_run = *runs.begin();
    {
      SCOPED_UMA_HISTOGRAM_LONG_TIMER("RenderTextHarfBuzz.GetFallbackFontTime");
      TRACE_EVENT1("ui", "RenderTextHarfBuzz::GetFallbackFont", "script",
                   TRACE_STR_COPY(uscript_getShortName(font_params.script)));
      const base::StringPiece16 run_text(&text[current_run->range.start()],
                                         current_run->range.length());
      fallback_found =
          GetFallbackFont(primary_font, locale_, run_text, &fallback_font);
    }

    if (fallback_found) {
      internal::TextRunHarfBuzz::FontParams test_font_params = font_params;
      if (test_font_params.SetRenderParamsOverrideSkiaFaceFromFont(
              fallback_font, fallback_font.GetFontRenderParams()) &&
          !FontWasAlreadyTried(test_font_params.skia_face,
                               &fallback_fonts_already_tried)) {
        ShapeRunsWithFont(text, test_font_params, &runs);
        MarkFontAsTried(test_font_params.skia_face,
                        &fallback_fonts_already_tried);
      }
    }

    // Remove the first run if not fully shaped with its associated fallback
    // font.
      if (!runs.empty() && runs[0] == current_run) {
        remaining_unshaped_runs.push_back(current_run);
        runs.erase(runs.begin());
      }
    }
    runs.swap(remaining_unshaped_runs);
    if (runs.empty()) {
      RecordShapeRunsFallback(ShapeRunFallback::FALLBACK);
      return;
    }

    std::vector<Font> fallback_font_list;
    {
      SCOPED_UMA_HISTOGRAM_LONG_TIMER("RenderTextHarfBuzz.GetFallbackFontsTime");
      TRACE_EVENT1("ui", "RenderTextHarfBuzz::GetFallbackFonts", "script",
                 TRACE_STR_COPY(uscript_getShortName(font_params.script)));
      fallback_font_list = GetFallbackFonts(primary_font);


    // Use a set to track the fallback fonts and avoid duplicate entries.
    SCOPED_UMA_HISTOGRAM_LONG_TIMER(
      "RenderTextHarfBuzz.ShapeRunsWithFallbackFontsTime");
    TRACE_EVENT1("ui", "RenderTextHarfBuzz::ShapeRunsWithFallbackFonts",
               "fonts_count", fallback_font_list.size());

    // Try shaping with the fallback fonts.
    for (const auto& font : fallback_font_list) {
      std::string font_name = font.GetFontName();

      FontRenderParamsQuery query;
      query.families.push_back(font_name);
      query.pixel_size = font_params.font_size;
      query.style = font_params.italic ? Font::ITALIC : 0;
      FontRenderParams fallback_render_params = GetFontRenderParams(query, NULL);
      internal::TextRunHarfBuzz::FontParams test_font_params = font_params;
      if (test_font_params.SetRenderParamsOverrideSkiaFaceFromFont(
            font, fallback_render_params) &&
        !FontWasAlreadyTried(test_font_params.skia_face,
                             &fallback_fonts_already_tried)) {
      ShapeRunsWithFont(text, test_font_params, &runs);
      MarkFontAsTried(test_font_params.skia_face,
                      &fallback_fonts_already_tried);
      }
      if (runs.empty()) {
        TRACE_EVENT_INSTANT2("ui", "RenderTextHarfBuzz::FallbackFont",
                           TRACE_EVENT_SCOPE_THREAD, "font_name",
                           TRACE_STR_COPY(font_name.c_str()),
                           "primary_font_name", primary_font.GetFontName());
        RecordShapeRunsFallback(ShapeRunFallback::FALLBACKS);

        return;
      }
    }

    for (internal::TextRunHarfBuzz*& run : runs) {
      if (run->shape.missing_glyph_count == std::numeric_limits<size_t>::max()) {
        run->shape.glyph_count = 0;
        run->shape.width = 0.0f;
      }
    }

    RecordShapeRunsFallback(ShapeRunFallback::FAILED);
    }

如果我没看错的话，它是先 `GetFallbackFont` 取一个字体，不管用才会 `GetFallbackFonts` 取全部挨个试。而 `GetFallbackFont` 我之前后面的代码没贴：

    // Try each font in the cache to find the one with the highest coverage.
    size_t fewest_missing_glyphs = text.length() + 1;
    const FallbackFontEntry* prefered_entry = nullptr;

    for (const auto& entry : cache_entry->second) {
      // Validate that every character has a known glyph in the font.
      size_t missing_glyphs = 0;
      size_t matching_glyphs = 0;
      size_t i = 0;
      while (i < text.length()) {
        UChar32 c = 0;
        U16_NEXT(text.data(), i, text.length(), c);
        if (entry.HasGlyphForCharacter(c)) {
          ++matching_glyphs;
        } else {
          ++missing_glyphs;
        }
      }

      if (matching_glyphs > 0 && missing_glyphs < fewest_missing_glyphs) {
        fewest_missing_glyphs = missing_glyphs;
        prefered_entry = &entry;
      }

      // The font has coverage for the given text and is a valid fallback font.
      if (missing_glyphs == 0)
        break;
      }

      // No fonts can be used as font fallback.
      if (!prefered_entry)
        return false;

      sk_sp<SkTypeface> typeface = GetSkTypefaceFromPathAndIndex(
          prefered_entry->font_path(), prefered_entry->ttc_index());
      // The file can't be parsed (e.g. corrupt). This font can't be used as a
      // fallback font.
      if (!typeface)
        return false;

      Font fallback_font(PlatformFont::CreateFromSkTypeface(
          typeface, font.GetFontSize(), prefered_entry->font_params()));

      *result = fallback_font;
      return true;
    }

它会针对某段 `text` 找一个 charset coverage 最高的 font，并不是针对某一个 glyph 找字体，所以就会受到 `text` 本身的影响。这可能是 google 为了 balance 速度想到的方法吧。

于是这个系列就结束了。也许下一篇会谈谈 `FcFontSort`？