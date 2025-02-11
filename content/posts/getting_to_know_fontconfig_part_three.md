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

我的问题解决了。下一步就是修改我的代码去了，顺便给大家一个去掉系统上已安装字体中 emoji 字符的 c 文件吧。因为 fontconfig 关于 `FcCharSet*`相关方法的 Public 函数里没有 `FcCharSetIterStart` 和 `FcCharSetIterNext`，没有办法 loop charset 里的 leaf 并打印。 也没有 `FcNameUnparseCharSet`，没有办法直接打印 `charset`。我是魔改了 `fontconfig.h`、`src/fcint.h`写出来的，需要你重新编译 fontconfig。

    #include<stdio.h>
    #include<fontconfig/fontconfig.h>

    int main(int argc, char **argv){
      FcFontSet* fs;
      FcPattern* pat;
      FcObjectSet* os;

      FcChar8* strpat = (FcChar8*)":lang=und-zsye";
      pat = FcNameParse(strpat);
      os = FcObjectSetBuild(FC_FAMILY, FC_CHARSET, FC_FILE, (char*)0);

      fs = FcFontList(0, pat, os);

      FcPatternDestroy(pat);
      FcObjectSetDestroy(os);

      FcCharSet* cs = FcCharSetCreate();
      int i;
      for(i=0; i<fs->nfont; i++){
        FcCharSet* tmp;
        if (FcPatternGetCharSet(fs->fonts[i], FC_CHARSET, 0, &tmp) != FcResultMatch) {
          goto nofont;
        }
        cs = FcCharSetUnion(cs, tmp);
        printf("%d\n", FcCharSetCount(cs));
      }
  
     FcFontSetDestroy(fs);
  
     FcPattern* pat1;
     FcFontSet* fs1;
     FcObjectSet* os1;
     
     if (argc == 1) {
         printf("you have to specify a font family to subtract!\n);
         goto nofont;
     }
     FcChar8* strpat1 = (FcChar8*)argv[1];
     pat1 = FcNameParse(strpat1);
     os1 = FcObjectSetBuild(FC_FAMILY, FC_CHARSET, FC_FILE, (char*)0);
     fs1 = FcFontList(0, pat1, os1);
  
     FcPatternDestroy(pat1);
     FcObjectSetDestroy(os1);
  
     FcCharSet* cs1 = FcCharSetCreate();
     for(i=0; i<fs1->nfont; i++){
        if (FcPatternGetCharSet(fs1->fonts[i], FC_CHARSET, 0, &cs1) != FcResultMatch) {
          goto nofont;
        }
     }
     FcFontSetDestroy(fs1);
  
     FcCharSet* cs2 = FcCharSetCreate();
     cs2 = FcCharSetIntersect(cs1, cs);
  
     FcStrBuf buf;
     FcChar8 init_buf[1024];
     FcStrBufInit(&buf, init_buf, sizeof(init_buf));
     
     if (FcNameUnparseCharSet(&buf, cs) && FcStrBufChar(&buf, '\0')) {
         printf("%s\n", buf.buf);
     } else {
         printf("charset (alloc error).\n");
     }
  
      FcStrBufDestroy(&buf);
      FcCharSetDestroy(cs);
      FcCharSetDestroy(cs1);
      FcCharSetDestroy(cs2);
      return 0;

    nofont:
      return 1;
    }

使用方法：下载 [fc-emoji-subtract.patch](https://gist.github.com/marguerite/e6eaab12010f6e7845d3ca351b253581)：

    git clone https://github.com/freedesktop/fontconfig
    patch -p1 < ../fc-emoji-subtract.patch
    
然后正常编译就可以，之后 `fontconfig/fc-emoji-subtract` 有一个 `fc-emoji-subtract`，用法：
     
     $ fc-emoji-subtract "Noto Sans CJK SC"
     20 23 2a 30-39 a9 ae 203c 2049 2122 2194-2199 24c2 25aa-25ab 25b6 25c0 2600-2603 260e 261d 262f 2640 2642 2660 2663 2665-2666 2668 267b 26a0 26bd-26be 2702 27a1 2934-2935 2b05-2b07 3030 303d 3297 3299 1f170-1f171 1f17e-1f17f 1f18e 1f191-1f19a 1f201-1f202 1f21a 1f22f 1f232-1f23a 1f250-1f251
你就可以针对这些 `charset` 自己写规则了。

## 其他我在第一部分抛出问题的答案

首先，`FcConfigSubstitute` 删除 charset leaf，如果你是显式的用，就是在 `FcListPatternMatchAny`阶段，因为 `config` 里无论你是 `assign` 还是 `assign_replace` 都认的。如果你是隐式的用，那就是在 `FcFileScanFontConfig` 的时候，但是必须用 `assign_replace`。

其次，`FcConfigSubstitue` 只会做 `<match target="pattern">`，如果你的第一个参数也就是 `config` 传 0 会做 `<match target="scan">`。这是我验证过的。而 `<match target="font">` 只会在一个地方做：`FcFontRenderPrepare`（可以搜 `FcMatchFont` 查到）。另外 Chromium 的 ` GetFontRenderParamsFromFcPattern` 是没有调用 `FcFontRenderPrepare` 的，它只会取字体默认的 aa，autohint, embedded_bitmap, hinting 和 rgba。~~所以你们批评它也没错。~~ **不做是对的，直接用缓存，把做 scan 的权力交给你自己**

下一篇，会谈谈双猫说的 Chromium 只返回一个 Fallback 字体的问题。敬请期待。
