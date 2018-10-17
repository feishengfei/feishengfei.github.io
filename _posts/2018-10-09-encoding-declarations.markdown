---
title: "Encoding declarations"
date: "2018-10-09 09:38:45 +0800"
category: work
tags: [python, ruby]
toc: true
excerpt: "About encoding on python & ruby"
---

## For Python

If a comment in the first or second line of the Python script matches the regular expression coding[=:]\s*([-\w.]+), this comment is processed as an encoding declaration; the first group of this expression names the encoding of the source code file. The encoding declaration must appear on a line of its own. If it is the second line, the first line must also be a comment-only line. The recommended forms of an encoding expression are

```python
# -*- coding: <encoding-name> -*-
```

## For Ruby
[参考](http://www.runoob.com/ruby/ruby-encoding.html)

Ruby 文件中如果未指定编码，在执行过程会出现报错：

```ruby
#!/usr/bin/ruby -w

puts "你好，世界！";
```

以上程序执行输出结果为：
```
invalid multibyte char (US-ASCII) 
```

以上出错信息显示了 Ruby 使用用 ASCII 编码来读源码，中文会出现乱码，解决方法为只要在文件开头加入 # -*- coding: UTF-8 -*-（EMAC写法） 或者 #coding=utf-8 就行了。

```
#!/usr/bin/ruby -w
# -*- coding: UTF-8 -*-
 
 puts "你好，世界！";
```
