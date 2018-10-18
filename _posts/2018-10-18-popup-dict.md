---
title: "popup dict"
date: "2018-10-18 23:43:44 +0800"
categories: 
  - work
  - efficiency
tags:
  - ubuntu
excerpt: Linux 下的划词翻译小工具 
classes: wide
---

## 来源

项目原始路径如下[popup-dict](https://github.com/bianjp/popup-dict)

作者：[bianjp](https://bianjp.com/)

作者对软件的[介绍](https://bianjp.com/posts/2017/12/14/announcing-popup-dict)

## 说明

![popup](https://raw.githubusercontent.com/bianjp/popup-dict/master/screenshots/popup.png)

功能特点

* 目前只支持英文->中文翻译，支持单词和短语
* 主要针对 Gnome 桌面环境，不保证其它环境下的正常使用
* 鼠标划词翻译，弹窗显示
* 智能处理选中内容（去除两端非英文字符、压缩空白字符、删除换行符等）
* 弹窗显示一段时间后自动关闭。若鼠标在弹窗中，延迟关闭
* 点击弹窗中链接可打开有道词典网页版

## 安装

`pip3 install popupdict`

## 其它

原始作者很贴心地制作了gnome插件[popup-dict switcher](https://extensions.gnome.org/extension/1349/popup-dict-switcher/), 亲测在gnome下的ubuntu18.04可以使用，且非常好用。

由于目前原作者尚未给快捷键自定义的切换开关，给出bash脚本如下，以方便通过绑定快捷键的方式进行开启、关闭、重启、切换操作。

```bash
#!/bin/sh

PS_KEYWORD=popup-dict
PS_KEYWORD0=python3
PROCESS=/usr/local/bin/popup-dict

case $1 in
	start)
		ps aux |grep -v grep | grep ${PS_KEYWORD} | grep ${PS_KEYWORD0} -q
			if [ $? -ne 0 ]; then
				${PROCESS} 2>&1 > /dev/null &
				export DISPLAY=:0.0 && /usr/bin/notify-send -i accessories-dictionary "On" "popup-dict on"
			fi
		;;

	stop)
		for pid in `pidof ${PS_KEYWORD0}`;do 
			grep -q ${PS_KEYWORD} /proc/${pid}/cmdline
			if [ $? -eq 0 ]; then
				export DISPLAY=:0.0 && /usr/bin/notify-send -i accessories-dictionary "Off" "popup-dict off"
				kill ${pid}
			fi
		done;
		;;

	switch)
		ps aux |grep -v grep | grep ${PS_KEYWORD} | grep ${PS_KEYWORD0} -q
			if [ $? -ne 0 ]; then
				#if offline, turn it on
				$0 start
			else
				#else online, turn it off
				$0 stop
			fi

		;;

	restart)
		$0 stop
		sleep 1
		$0 start
		;;
esac

exit 0

```

<br>
This work is licensed under a **[Attribution 4.0 International license](https://creativecommons.org/licenses/by/4.0/)**. ![Attribution 4.0 International](https://licensebuttons.net/l/by/4.0/88x31.png)
{: .notice}


