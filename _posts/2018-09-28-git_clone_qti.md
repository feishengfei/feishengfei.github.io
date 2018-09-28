---
title:  "关于从qualcomm官网clone代码"
date:   2018-09-28 17:18:39 +0800
---
需要设置http.followRedirects为true。
```
frankzhou@fx:~/github$ git config --global http.followRedirects true
frankzhou@fx:~/github$ git clone https://chipmaster2.qti.qualcomm.com/home2/git/broadmobi/mdm9607-le-2-2_amss_standard_oem.git
Cloning into 'mdm9607-le-2-2_amss_standard_oem'...
Username for 'https://chipmaster2.qti.qualcomm.com': zhouxiang@broadmobi.com
Password for 'https://zhouxiang@broadmobi.com@chipmaster2.qti.qualcomm.com':
warning: redirecting to https://chipmaster2.qti.qualcomm.com/home/git/broadmobi/mdm9607-le-2-2_amss_standard_oem.git/
remote: Counting objects: 106627, done.
remote: Compressing objects: 100% (86905/86905), done.
...
```
