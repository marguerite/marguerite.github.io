---
title: "深入理解 Linux fontconfig 第二部分"
date: 2022-11-25T19:17:09+08:00
draft: false
---
这是深入理解 Linux fontconfig 系列的第二部分，第一部分请[移步](https://marguerite.su/posts/getting_to_know_fontconfig_part_one/)。

前面说到了 `FcConfigSubstitute`这个函数，它实际封装了：

    FcConfigSubstituteWithPat (FcConfig    *config,
			   FcPattern   *p,
			   FcPattern   *p_pat,
			   FcMatchKind kind)
    {
这不过第二个 p_pat 是 0。

整个 FcConfigSubstituteWithPat 函数比较长，就不全贴出来了。具体在[这里](https://github.com/freedesktop/fontconfig/blob/master/src/fccfg.c)。

它先是获取了 config：

    config = FcConfigReference (config);
 
然后针对 FcMatchPattern 这个 kind 先往 pattern *p 里加入了 lang 和 prgname 元素。接着就是针对从 config 里取到的某个 kind 的 RuleSet 挨个应用到 pattern。

## FcConfigReference

实际上只有给它传 0 才会做一大堆初始化，不然就会使用你自己的 config。前面说过 Chromium 的 config 就是它自己做的：

    FcConfig* config = GetGlobalFontConfig();
    
实际上包括 fontconfig 自己在内都是直接传 0 让它做初始化的。初始化最重要的函数是：

    config = FcInitLoadConfigAndFonts ();

这个函数在 [fcinit.c](https://github.com/freedesktop/fontconfig/blob/6f27f42e6140030715075aa3bd3e5cc9e2fdc6f1/src/fcinit.c)。实际封装了 `FcInitLoadOwnConfigAndFonts(NULL) `。这个函数长这样：

    FcConfig *
    FcInitLoadOwnConfigAndFonts (FcConfig *config) {
        config = FcInitLoadOwnConfig (config);
        if (!config)
	      return 0;
        if (!FcConfigBuildFonts (config)) {
	      FcConfigDestroy (config);
	      return 0;
        }
        return config;
    }
    
它首先是使用 `FcInitLoadOwnConfig` 去加载默认 config，然后看看默认 config 能否 `FcConfigBuildFonts`，如果能就返回。

`FcInitLoadOwnConfig` 这个函数非常大，但是由于前面传过来的是 NULL, 理解起来反而更简单。它其实就是生成了一份默认的什么都没有但是能相应各种请求的 FcConfig，内容就是这样：

     <fontconfig>
	 FC_DEFAULT_FONTS
	 <dir prefix=\"xdg\">fonts</dir>
	 <cachedir>" FC_CACHEDIR "</cachedir>
	 <cachedir prefix=\"xdg\">fontconfig</cachedir>
	 <include ignore_missing=\"yes\">" CONFIGDIR "</include>
	 <include ignore_missing=\"yes\" prefix=\"xdg\">fontconfig/conf.d</include>
	 <include ignore_missing=\"yes\" prefix=\"xdg\">fontconfig/fonts.conf</include>
	</fontconfig>

重要的反而是 `FcConfigBuildFonts`。而后者里面最重要的函数是 `FcConfigAddDirList `。接着最重要的是 ` FcDirCacheRead`->`FcDirCacheScan`->`FcDirScanConfig`-> `FcFileScanConfig `-> `FcFileScanFontConfig`。

## FcFileScanFontConfig

这个函数是一个非常重要的底层函数，因为它有：

    if (!FcFreeTypeQueryAll (file, -1, NULL, NULL, set))
    
去真正的扫描字体文件，并且对扫描出的 FontSet 逐个有：

    if (config && !FcConfigSubstitute (config, font, FcMatchScan))
    
也就是应用 `<match target="scan">` 规则。而我们的 charset minus 应用的正是 scan 规则。而后面：

	if (FcDebug() & FC_DBG_SCANV)
	{
	    printf ("Final font pattern:\n");
	    FcPatternPrint (font);
	}
	
也给了我们一个搜索点 `Final font pattern`来确定 `scan` 规则是否应用成功。但是由于没有 `FcCharsetPrint`只有 `FcPatternPrint`，我们很难精确的在输出中看到 charset 的。

至此我们的函数理解阶段就到此为止了。因为我们能够确定：

**只要传递给 FcConfigSubstitute 中的 config 是 0，那么一定能够得到应用了 charset minus 方法后的字体。**

## 结论

首先，我们之前一直在说 fontconfig 的 scan 阶段生成的是一个`无序列表`，实际上不对。

它的 directory 是严格按照 `config->fontDirs`的顺序来的，而每个文件夹中字体文件的扫描顺序是：

     /*
     * Sort files to make things prettier
     */
    qsort(files->strs, files->num, sizeof(FcChar8 *), cmpstringp);

它实际上是 [strcmp](https://www.programiz.com/c-programming/library-function/string.h/strcmp) 的顺序。可以在 fcdir.c 里看到。

其次， `scan->pattern->font` 的 match 顺序还是正确的。

再次，通过 FcFontList 带 `objectset` 的方式和 `FcConfigSubstitute`传空 config（第一个参数为 0）的方式都可以获取到应用了 charset minus 方法的字体。

最后，使用以上方法，字体缓存中就是应用了 charset minus 方法的字体，最终返回的 font_pattern 的 charset 里也没有被减掉的字符。但是如果没有使用以上方法，`FcInitLoadOwnConfig` 就会跳转到下一段去，解析它提供的 config，那么比如缓存文件夹就可能会与系统的不同，那它缓存文件怎么来的就不好说了...再换句话说，config 用没用系统的都不好说呢。

## 后续

原理部分讲完了。下一篇我们就该继续前面的 Chromium 话题了。因为 Chromium 给 `FcConfigSubstitute` 传的 config 不是 0 而是它自己的：

    FcConfig* config = GetGlobalFontConfig();
    

敬请期待。
