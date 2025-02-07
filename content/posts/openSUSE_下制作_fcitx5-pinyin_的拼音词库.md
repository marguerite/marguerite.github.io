---
title: "openSUSE 下制作 fcitx5-pinyin 的拼音词库"
date: 2025-02-07T00:00:00+08:00
draft: false
---
年前换了 am5 平台，把 Windows 7 搞坏了（B650m 主板没有 PS/2，amd 又没有 Win7 下的 USB 驱动），只能在 openSUSE 处理一些工作上的事情。我面临的问题是：输入一些奇奇怪怪的国内公司名字的时候，搜狗输入法NG版本（基于 CPIS），记忆比较快，但是它有几个硬伤解决不了，比如候选词面板上的数字上屏，有时候会跳，输入3上的是第5个字; 比如使用输入法切换到英文输入模式就切换不回来了;比如一些基本的汉字打不出来，比如覆盖的覆字。而 fcitx5 呢，记忆又太慢，我不得不一次次的去敲比如“鑫永俪”。于是我想到了一个办法，就是把我常用的公司名字制作成拼音词库。

fcitx5 的 libime 提供了 libime-pinyindict 工具用来把文本转为词库，要求的文本格式是这样的：

	鑫永俪	xin'yong'li	0

于是我需要的就是把 excel 格式的客户名单分词，再标注拼音就可以了（词频默认是 0）。我的客户名单是这样的：

	91110000MA0UXXXXXX 北京市鑫永俪金融服务有限责任公司 李华梅 13800138000

所以先需要一段 python 脚本去处理 excel

	#!/usr/bin/env python3
	
	import xlrd
	import jieba
	import sys
	
	''' 输出 Excel 中的中文公司名和法人名 '''
	def get_chinese_words_from_excel(f):
		book = xlrd.open_workbook(f)
		sheet = book.sheet_by_index(0)
		res = []
		
		for i in range(sheet.nrows):
		    # 9 开头的是公司
			if not sheet.cell_value(i, 0).startswith('9'):
				continue
				
			for j in range(sheet.ncols):
				if j not in (1, 2):
					continue
				if len(sheet.cell_value(i, j).strip()) > 0:
				    print(sheet.cell_value(i, j).strip())
					seg_list = jieba.cut(sheet.cell_value(i, j).strip())
					for s in seg_list:
						if s not in res:
						    print(s)
							res.append(s)
							
	   return res
	   
	if len(sys.argv) > 1:
		get_chinese_words_from_excel(sys.argv[1])
		
这样就得到了：

	北京市鑫永俪金融服务有限责任公司
	北京
	北京市
	鑫
	永
	俪
	金融
	服务
	金融服务
	有限
	责任
	公司
	有限责任
	有限责任公司
	李华梅
	李
	华梅
	
这里有一个问题，它发现组不了词的（往往是我要的），会返回单字。

于是进一步处理，把长度小于 2 的，与上一行拼接，同时生成拼音：

	#!/usr/bin/env python3
	
	from pypinyin import lazy_pinyin
	
	arr = []
	with open('text.txt', 'r') as f:
	    last_line = ''
		for line in f.readlines():
			if len(line.strip()) > 2:
				last_line = line.strip()
			else:
			    line = last_line + line.strip()
			    last_line = line
			if line.strip() not in arr:
			    arr.append(line.strip())
			    
	str = ''
	for a in arr:
		str += a + '\t\' + "'".join(lazy_pinyin(a)) + '\t0\n'

	f1 = open('dict.txt', 'w')
	f1.write(str)
	f1.close()
	
然后再使用：

	/usr/bin/libime_pinyindict dict.txt company.dict
	mkdir -p ~/.local/share/fcitx5/pinyin/dictionaries
	cp -r company.dict ~/.local/share/fcitx5/pinyin/dictionaries

重启 fcitx5 生效。

总结一下，openSUSE 下面制作 fcitx5 词库还是比较简单的，第一步分词，第二步添加拼音和词频。我第二步的脚本是可以通用的，只需要把那个 text.txt 处理好就可以。而我的 text.txt 的处理方式比较特殊，因为我是从 excel 文件来的。后续我做行业词库的时候还改脚本把 word 文档喂给了它。如果想要自制词库，需要处理好语料的获取。分词方面，我用的是 jieba, 准确率不算很高，你也可以试试 pkuseg, 这两个包在 openSUSE 下面都是有的。
