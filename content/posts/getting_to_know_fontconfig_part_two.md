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
 
然后针对 FcMatchPattern 这个 kind 先往 pattern p 里加入了 lang 和 prgname 元素。接着就是针对从 config 里取到的某个 kind 的 RuleSet 挨个应用到 pattern。

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
    
也就是应用 `<match target="scan">` 规则。而我们的 charset minus 应用的正是 scan 规则。

这里要特殊说一下，就是以上流程是在假设没有 `cache` 的情况下才会发生的，如果有 `cache`，那么到了 `FcDirCacheRead` 那步就是读取 `cache` 了。所以上面分析的过程相当于：

     fc-cache && fc-match blabla

至此我们的函数理解阶段就告一段落了。因为我们能够确定：

**只要传递给 FcConfigReference 中的 config 是 0，那么一定能够得到应用了 `<match target="scan">` 后的字体。**

## 验证 FcConfigReference

魔改了一下依云的代码：

    #include<stdio.h>
    #include<fontconfig/fontconfig.h>

    int main(int argc, char **argv){
      FcFontSet* fs = FcFontSetCreate();
      FcPattern* pat;
      
      FcChar8* strpat = (FcChar8*)":lang=zh";
      pat = FcNameParse(strpat);

      FcConfigSubstitute(0, pat, FcMatchPattern);
      FcDefaultSubstitute(pat);

      FcResult ret;
      FcFontSet* font_patterns = FcFontSort(0, pat, FcFalse, 0, &ret);

      int j;
      for (j = 0; j < font_patterns->nfont; j++) {
        FcPattern* font_pattern = FcFontRenderPrepare(NULL, pat, font_patterns->fonts[j]);
        if (font_pattern) {
          FcFontSetAdd(fs, font_pattern);
        }
      }

      FcFontSetSortDestroy(font_patterns);
      FcPatternDestroy(pat);

      FcChar8 *family;
      FcChar8 *file;
      FcCharSet* cs;
      FcChar32 ch;
      FcUtf8ToUcs4((FcChar8*)"™", &ch, 3);
      
      int i;
      for(i=0; i<fs->nfont; i++){
        if(FcPatternGetCharSet(fs->fonts[i], FC_CHARSET, 0, &cs) != FcResultMatch){
          fprintf(stderr, "no match\n");
          FcPatternPrint(fs->fonts[i]);
          goto nofont;
        }
        if(FcPatternGetString(fs->fonts[i], FC_FAMILY, 1, &family) != FcResultMatch)
          if(FcPatternGetString(fs->fonts[i], FC_FAMILY, 0, &family) != FcResultMatch)
            goto nofont;
          printf("[%d] %s ", i, (char *)family);
        if(FcPatternGetString(fs->fonts[i], FC_FILE, 0, &file) != FcResultMatch)
          goto nofont;
        printf("(%s): ", (char *)file);
        if(FcCharSetHasChar(cs, ch)){
          puts("Yes");
        } else {
          puts("No");
        }
    }

    FcFontSetDestroy(fs);

    return 0;

    nofont:
       return 1;
    }

编译运行：

    gcc -o test1 test1.c -lfontconfig
    ./test1 | grep "Noto Sans CJK SC"
结果：

    [0] Noto Sans CJK SC (/usr/share/fonts/truetype/NotoSansCJKsc-Regular.otf): Yes

 居然还在！
 
其实这不奇怪：

首先我们只是知道了给 `FcConfigReference` 传 0 会得到应用 `<match target="scan"` 的结果，结果里面是否成功去掉了 charset leaf，我们没有验证。

其次，前面我们说 `FcFontList`的时候提到的 config 其实也是传了 0 的，但最后没应用 `objectset` 得到的结果也是还有 `Noto Sans CJK SC` ，这是符合 `FcFontList` 的，因为你并没有精确的要 `:charset=0x2122`，但是后面 `FcPatternGetCharset` 的结果里含有 `charset=0x2122` 这就不对了。

第三，也有可能是 `FcDefaultSubstitute`、`FcFontSort`、`FcFontRenderPrepare`这其中一个调整过 charset，把它给恢复了。但不可能。我在调用 `FcFontSort` 的时候 `trim` 选项给的是 `FcFalse`，不会改。`FcDefaultSubstitute` 大家都摸得很透了，就是添加一些默认的 slant、weight 之类的，`FcFontRenderPrepare` 通篇也都在围绕 lang、size 这些做文章，根本就没碰 charset。
 
也就是说：

    if (config && !FcConfigSubstitute (config, font, FcMatchScan))
    
这里可能是应用了，但没有应用成功。那我们就只能魔改这个函数让它吐前后的 charset 来验证了。

## 魔改 FcFileScanFontConfig

我们给 `FcFileScanFontConfig` 函数添加以下代码：

    static FcBool
    FcFileScanFontConfig (FcFontSet		*set,
		      const FcChar8	*file,
		      FcConfig		*config)
    {
        int		i;
        FcBool	ret = FcTrue;
        int		old_nfont = set->nfont;
        const FcChar8 *sysroot = FcConfigGetSysRoot (config);

        + FcChar8* family;
        + FcCharSet* cs;
        + FcCharSet* cs1;
        + FcChar32 ch;
        + FcUtf8ToUcs4((FcChar8*)"™", &ch, 3);
然后在：

    if (config && !FcConfigSubstitute (config, font, FcMatchScan))
 
前后分别加上：

    if (FcDebug() & FC_DBG_SCANV)
    {
      FcPatternGetString(font, FC_FAMILY, 0, &family);
      printf ("%s\n", family);
      printf("before match scan: ");
      FcPatternGetCharSet(font, FC_CHARSET, 0, &cs);
      if (FcCharSetHasChar(cs, ch)) {
        printf("Yes\n");
      } else {
        printf("No\n");
      }
      printf("charset count: %d\n", FcCharSetCount(cs));
    }
很简单吧，就是打印它前后进行 FcCharSetHasChar 的结果。然后编译，运行：

    su
    FC_DEBUG=256 fc-cache -f > fc-cache.txt
结果让人十分沮丧：

    Noto Sans CJK SC
    before match scan: Yes
    charset count: 44810
    after match scan: Yes
    charset count: 44810

## 为什么？

到这儿其实已经可以定义为一个 BUG 了。宣称 minus charset 最后却没有减。但我还是有兴趣继续 debug 一下 FcMatchScan 是怎么进行的。

留个悬念吧，其实它不是 bug。详情见下篇分解。

## 初步结论

首先，我们之前说 fontconfig 的 scan 阶段生成的是一个`无序列表`，实际上不对。

它的 directory 是严格按照 `config->fontDirs`的顺序来的，而每个文件夹中字体文件的扫描顺序是：

     /*
     * Sort files to make things prettier
     */
    qsort(files->strs, files->num, sizeof(FcChar8 *), cmpstringp);

它实际上是 [strcmp](https://www.programiz.com/c-programming/library-function/string.h/strcmp) 的顺序。可以在 fcdir.c 里看到。

其次， `scan->pattern->font` 的 match 顺序还是正确的。

再次，通过 `pattern` 中带 `:charset=0x2122` 的方式可以获取到应用了 charset minus 方法后的字体。`FcConfigSubstitute` 传 0 这种方法暂时不行，不行的原因是 `<match target="scan">` 虽然应用了，但是没有影响到 pattern。

最后，理论上，通过上面两种方法，字体缓存中就是应用了 charset minus 方法的字体，最终返回的 font_pattern 的 charset 里也没有被减掉的字符。但是如果使用 `FcConfigSubstitute` 方法却不传 0，`FcInitLoadOwnConfig` 就会跳转到下一段去，解析它提供的 config，那么比如缓存文件夹就可能会与系统的不同，那它缓存文件怎么来的就不好说了...再换句话说，config 用没用系统的都不好说呢。

## 谈谈 Chromium 的 config

原理部分讲完了。我们继续前面的 Chromium 话题。因为 Chromium 给 `FcConfigSubstitute` 传的 config 不是 0 而是它自己的：

    FcConfig* config = GetGlobalFontConfig();

而这个 GetGlobalFontConfig() 是在 [ui/gfx/linux/fontconfig_util.cc](https://github.com/chromium/chromium/blob/main/ui/gfx/linux/fontconfig_util.cc) 里定义的：

    GlobalFontConfig() {
    // Without this call, the FontConfig library gets implicitly initialized
    // on the first call to FontConfig. Since it's not safe to initialize it
    // concurrently from multiple threads, we explicitly initialize it here
    // to prevent races when there are multiple renderer's querying the library:
    // http://crbug.com/404311
    // Note that future calls to FcInit() are safe no-ops per the FontConfig
    // interface.
    FcInit();

    // Increment the reference counter to avoid the config to be deleted while
    // being used (see http://crbug.com/1004254).
    fc_config_ = FcConfigGetCurrent();
    FcConfigReference(fc_config_);

    // Set rescan interval to 0 to disable re-scan. Re-scanning in the
    // background is a source of thread safety issues.
    // See in http://crbug.com/1004254.
    FcBool result = FcConfigSetRescanInterval(fc_config_, 0);
    DCHECK_EQ(result, FcTrue);
    }

实际上是调用了 `FcConfigGetCurrent`->`FcConfigEnsure`，最终还是会使用系统上的配置的，跟 0 没什么两样。只是它不再去 rescan 了。也就是说你每次改完 fontconfig 配置，恐怕得重启 chromium 并清空缓存而不是重开网页才能看见效果。
