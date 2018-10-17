---
title: "写在bashrc中的bitbake帮助工具"
date: "2018-10-09 14:00:28 +0800"
category: work
tags: [bitbake, yocto]
classes: wide
excerpt: 本文提供了两个预定义的bash函数，方便在使用bitbake编译时，截取关键的NOTE开头的信息，并将安整的日志保存在`~/log/`中，以供日后查询。
---

如题，bitbake在开启`-vDD`之后的输出较多，且开启调试编译之后如何保存这些日志也是较为麻烦的，为了提高效率，特提供写在bashrc中的两个工具如下。

```bash
rb(){
    DATE=
    #DATE=_`date +%Y%m%d_%H%M`
    if [ -n `which bitbake` ]; then
        if [ ! -d ~/log/ ]; then
            mkdir ~/log/ -p
        fi
        if [ 2 -eq $# ]; then
            rebake $1 -c $2 -vDD 2>&1 |tee ~/log/bb_${1}__${2}${DATE}.log | grep -e ^NOTE -e ^ERROR|grep -e \ do_[a-z_]*[\ \:] -e Failed -e ^ERROR -e "not found"
        elif [ 1 -eq $# ]; then
            rebake $1 -vDD 2>&1 |tee ~/log/bb_${1}__default${DATE}.log | grep -e ^NOTE -e ^ERROR|grep -e \ do_[a-z_]*[\ \:] -e Failed -e ^ERROR -e "not found"
        else
            echo -e 'rb <RECIPE_NAME> <TASK_NAME>'
            echo -e '[usage] rb for short to rebake your RECIPE or RECIPE+TASK with log supported\n'
        fi
    fi
}

bb(){
    DATE=
    #DATE=_`date +%Y%m%d_%H%M`
    if [ -n `which bitbake` ]; then
        if [ ! -d ~/log/ ]; then
            mkdir ~/log/ -p
        fi
        if [ 2 -eq $# ]; then
            bitbake $1 -c $2 -vDD 2>&1 |tee ~/log/bb_${1}__${2}${DATE}.log | grep -e ^NOTE -e ^ERROR |grep -e \ do_[a-z_]*[\ \:] -e Failed -e ^ERROR -e "not found"
        elif [ 1 -eq $# ]; then
            bitbake $1 -vDD 2>&1 |tee ~/log/bb_${1}__default${DATE}.log | grep -e ^NOTE -e ^ERROR |grep -e \ do_[a-z_]*[\ \:] -e Failed -e ^ERROR -e "not found"
        else
            echo -e 'bb <RECIPE_NAME> <TASK_NAME>'
            echo -e '[usage] bb for short to bitbake your RECIPE or RECIPE+TASK with log supported\n'
        fi
    fi
}
```

其中rb代表rebake，即cleanall指定recipe之后指定某个recipe的task进行编译，或不指定task而进行默认编译；相应的，bb代表bitbake，即不进行cleanall而直接编译。

<br>
This work is licensed under a **[Attribution 4.0 International license](https://creativecommons.org/licenses/by/4.0/)**. ![Attribution 4.0 International](https://licensebuttons.net/l/by/4.0/88x31.png)
{: .notice}
