---
title: "openSUSE Tumbleweed 上启用 SELINUX"
date: 2025-09-20T10:37:00-08:00
draft: false
---

论坛的服务器是 tumbleweed，一直以来想启用 selinux 增强一下安全性，但最早 openSUSE 是 apparmor 系的（其实什么系都无所谓，因为个人电脑没启用过这些），就很犹豫，有 selinux 相关的包但是不敢用。现在官方已经切换到 selinux 了，而且新版本的 tumbleweed ISO 默认就是启用 selinux 的，我决定冲一次。

#### 安装和启用

    sudo zypper in -t pattern selinux
    
就会安装全部 selinux 相关的包，然后进 YaST 引导加载器模块，把 kernel parameter 的最后加上：

	security=selinux selinux=1
	
保存让 YaST 重新生成 initrd 即可。

最后编辑 /etc/selinux/config

	SELINUX=permissive
	SELINUXTYPE=targeted
	
刚开始一定不能是 `enforcing`，不然可能会被卡在外面进不去系统。最后：

	sudo touch /.autorelabel
	
重启自动标记系统内全部文件即可。

重启之后，看看有没有 avc（Access Vector Cache，访问矢量高速缓存）错误和 selinux error。

	sudo ausearch -m avc,user_avc,selinux_err,user_selinux_err -ts boot
	
结果出现一个：

	avc: denied { relabelto } for pid=2128 common="snapperd" name="info.xml" dev="sda4" ino=6425
	 scontext=scontext=system_u:system_r:snapperd_t:s0-s0:c0.c1023
	 tcontext=system_u:object_r:snapperd_data_t:s0 tclass=file
	 
#### 调试

好多年前我把 AppArmor 翻译为“应用装甲”的时候也是了解过强制访问控制系统的知识的，selinux 就是把文件的访问权限按照“用户：角色：数据类型：级别：分类”去强化一下。如果任意一项不满足条件，就拒绝访问。

这个提示的意思就是 relabel 这个 info.xml 的过程中出错了。

用 audit2why 来分析一下：

	Missing type enforcement (TE) allow rule。
	
然后就是让我用 audit2allow 来生成一个模块。但是 openSUSE 的人说了轻易不要生成模块，因为可能把权限放太松了，导致失去意义。

#### 暴力排查

我想，既然是 snapper 快照的问题，可不可以找到这个 info.xml 是哪个快照的，如果不是最后一个快照，直接把这个快照删了会不会解决问题。因为快照很多的，如果真的是缺失 TE 规则的话，也应该给我刷出一屏幕才对，不会只是针对个别的快照。

先用 `df` 命令确定下 sda4 是哪里，我这边是 / 也就是根目录，同时 snapper 也告诉我是跟什么程序有关的，但是 /.snapshots 下面有好多快照，我怎么知道是哪个快照的 info.xml 除了问题呢？

	sudo find /.snapshots -type f -name "info.xml" -exec ls -i {} \;
	
这段 find 命令的意思是在 /.snapshots 目录下查找名为 info.xml 的文件，然后对它调用 `ls -i` 显示其 inode。

	6425 /.snapshots/1130/info.xml
	
跟上面提示的 `ino` 对上了。接下来查看它现在的 MAC 属性：

	ls -Z /.snapshots/1130/info.xml
	system_u:object_r:snapperd_data_t:s0
	
咦？跟提示里的目标属性（tcontext）是一致的，说明成功了，原来是一场误报。把 permissive 调成 enforcing，重启，进入桌面了。

PS：如果你的情况不是我的情况，可不能贸然把 permissive 调成 enforcing，可以试试不修改先重启一次，错误会不会再出现。

