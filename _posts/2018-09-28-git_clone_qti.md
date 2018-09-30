---
title:  "About how to git clone source code form qualcomm official site"
date:   2018-09-28 17:18:39 +0800
layout: default
---

Before `git clone`, you need to set *http.followRedirects* as *true*.

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
