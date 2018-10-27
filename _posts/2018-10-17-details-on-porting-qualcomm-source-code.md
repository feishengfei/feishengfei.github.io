---
title: "Details on porting qualcomm source code"
date: "2018-10-17 09:53:53 +0800"
toc: true
classes: wide
excerpt: 本文介绍了从高通官网下载原码并同步的详细过程，重启记录了从repo折叠到单一git仓时需要修改的gitignore文件，以备下次更新基线时参考。
categories:
  - work
tags:
  - qualcomm
  - git
classes: wide
toc: true
---

## 从qualcomm官网选择你感兴趣的平台版本

![select_source_code_you_want_to_import](/assets/images/qualcomm/how_to_port_00_select_source_code_you_want_to_import.png)

## 通过**git clone**下载或直接下载压缩包

你可以选择相应的分支，并选择对应的压缩格式，确定之后在你的帐户的download下面会生成相应的下载链接。

或者你可以以`git clone`克隆整个仓库。进行之前，确保配置**http.followRedirects**为**true**

```
frankzhou@fx:~/github$ git config --global http.followRedirects true
frankzhou@fx:~/github$ git clone https://chipmaster2.qti.qualcomm.com/home2/git/YOUR_COMPANY/mdm9607-le-2-2_amss_standard_oem.git
Cloning into 'mdm9607-le-2-2_amss_standard_oem'...
Username for 'https://chipmaster2.qti.qualcomm.com': YOUR_USERNAME@YOUR_DOMAIN
Password for 'https://YOUR_USERNAME@YOUR_DOMAIN@chipmaster2.qti.qualcomm.com':
warning: redirecting to https://chipmaster2.qti.qualcomm.com/home/git/YOUR_COMPANY/mdm9607-le-2-2_amss_standard_oem.git/
remote: Counting objects: 106627, done.
remote: Compressing objects: 100% (86905/86905), done.
...
```

## 从根目录按核心模块(分文件夹进行)进行第一次拆包提交

本次提交确保在同步之前进行。

* 在各个核心模块文件夹内先执行`git init`
* 将根目录下的.gitattribute拷贝到各个模块的文件夹下
* `git add`之前不要创建.gitignore忽略文件，因为从高通官网的初次下载可能包含一些中间依赖的二进制包，比如'.o'或'.pyc'等二进制文件，这里确保这些文件可以被`git add`添加入stagging中
* 在上一步进行`git add`之后，本步再创建.gitignore文件(除过apps_proc以外)，确保.pyc/.swp/.swo/.svn/tags文件被忽略
* 执行`git commit`提交


## 通过build ID与syncbuild.sh完成repo sync并进行编译

进行本步操作前，确保[repo](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/)在PATH中。

本步仅在apps_proc下进行，以同步来自CAF(Code Aurora Forum)的开源包。执行前先在根目录中的about.html中确定目前的BUILDID，之后执行`./syncbuild.sh YOUR_BUILD_ID`，以进行repo同步与首次编译。

## 修改gitignore文件进行第二批次的提交

本步仅在apps_proc下进行，涉及到的.gitignore文件如下。

> ./system/core/.gitignore

修改:忽略/oe-logs, /oe-workdir

```
#oe-logs, oe-workdir add by zhouxiang
/oe-logs
/oe-workdir
```

> ./poky/build/downloads/.gitignore

新建:忽略downloads下的空文件夹

```
#all added by zhouxiang
/*.done
/git2
/base
/wlan
/audio
/external
/qcom-opensource
/mdm9607
/telux
/mm-video-oss
/frameworks
/vendor
/telephony
/graphics
```

> ./poky/.gitignore

修改:排除downloads与conf。排除downloads是为下将初次编译涉及的远程下载包提交上去以节省重新编译时的下载时间；排除conf是因为conf下有自已独立的.gitignore文件。

```
#exclude following 2 directories by zhouxiang
!/build/conf
!/build/downloads
```

> ./filesystems/mtd-utils/.gitignore

修改:排除一些原代码目录与目标二进制重名的目录。

```
JitterTest
#checkfs masked by zhouxiang, which binary & source directory is consistent
#checkfs
free_space

...

makefiles
#mkfs.ubifs masked by zhouxiang, which binary & source directory is consistent
#mkfs.ubifs
mkvol_bad

...

/ltmain.sh
#m4 masked by zhouxiang, which binary & source directory is consistent
#/m4/
/missing
```

> ./external/open-avb/daemons/gptp/windows/.gitignore

修改:排除*.user文件

```
*.[Oo]bj
#masked by zhouxiang
#*.user
*.aps
```

> ./external/open-avb/examples/gstreamer-avb-plugins/gst-plugin/.gitignore

修改:排除一些被忽略的文件

```
#aclocal.m4 masked by zhouxiang
#aclocal.m4
autom4te.cache
autoregen.sh
#following 2 config prefixed files masked by zhouxiang
#config.*
#configure
libtool
#following 6 files masked by zhouxiang
#INSTALL
#Makefile.in
#depcomp
#install-sh
#ltmain.sh
#missing
stamp-*
my-plugin-*.tar.*
*~
```
> ./external/open-avb/.gitignore

修改:排除build文件夹

```
#build masked by zhouxiang
#build/
daemons/gptp/linux/build/obj/*.o
```

> ./external/libunwind/.gitignore

修改: 排除以下被忽略的文件

```
#following 3 headers are commented by zhouxiang
#include/config.h
#include/config.h.in
#include/libunwind-common.h
```

> ./.gitignore

新建:最终在apps_proc下建立顶级忽略文件如下

```
#added by zhouxiang
#python compiled binary files
*.pyc

#vi related
*.swp
*.swo

#exclude svn
.svn

#exlcude ctags
tags

#mtd-utils
filesystems/bin/target/mtd-utils/checkfs/*
filesystems/bin/target/mtd-utils/fstests/*
filesystems/bin/target/mtd-utils/ubi-tests/*

#YOCTO related
poky/build/bitbake.lock
poky/build/cache/
poky/build/sstate-cache/
poky/build/tmp-glibc/

#download related
poky/build/downloads/git2/
poky/build/downloads/kernel-tests/
poky/build/downloads/qcom-opensource/kernel/
poky/build/downloads/rsdlclient/
```

在完成以上添加/修改.gitignore的工作之后，再进行`git add`与`git commit`，并进行`git clone`与编译验证sync之后的文件是否有效地提交进仓库。如果验证过程中出现错误，需根据实际情况重新调整忽略再次reset并重新提交/克隆/编译/验证。

## 完成最小修改后的第三次提交

本步根据具体的硬件配置进行最小修改，并编译烧写验证，验证无误之后完成本次提交。

<br>
This work is licensed under a **[Attribution 4.0 International license](https://creativecommons.org/licenses/by/4.0/)**. ![Attribution 4.0 International](https://licensebuttons.net/l/by/4.0/88x31.png)
{: .notice}
