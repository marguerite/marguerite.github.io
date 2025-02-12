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

content_shell_build.sh:

    #!/bin/sh
    . ./set_environment.sh
    autoninja -C ./src/out/Debug content_shell
    
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
    ./src/out/Debug/content_shell --no-sandbox --enable-logging=stderr --vmodule="\*/fonts/\*=4" --v=-3 > log.txt 2>&1 $1

--vmodule  的意思是只要路径中含有 fonts 的日志，--v=-3 的意思是压制其他日志。跑一下：

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
    for (int i = 0; curr_family; curr_family = curr_family->Next()) {
        DVLOG(0) << " current font_family is " << curr_family->FamilyName();
        i++;
    }

然后重新编译一下 content_shell 再看。

注意：content_shell_wrapper.sh 里面的 loglevel 要从 4 改为 0。loglevel 是输出这个 level 及以下的，所以数字越小，内容越少。另外，虽然只改动了一个文件，但重新 LINK content_shell 的时间非常长，所以真正使用的时候，最好一次性把需要 DVLOG 输出的地方都改好再重新编译 content_shell。

运行结果如下：

    [18085:18085:0212/111541.814978:INFO:harfbuzz_shaper.cc(754)] fontdescription in HarfBuzzShaper::ShapeSegment
    [18085:18085:0212/111541.815081:INFO:harfbuzz_shaper.cc(756)]  current font_family is "Segoe UI" round:0
    [18085:18085:0212/111541.815120:INFO:harfbuzz_shaper.cc(756)]  current font_family is "Roboto" round:1
    [18085:18085:0212/111541.815152:INFO:harfbuzz_shaper.cc(756)]  current font_family is "sans-serif" round:2

当然了，以上只是示例，真正距离能够产出完整的字体调试输出还需要修改好多地方。但总算迈出了第一步。



