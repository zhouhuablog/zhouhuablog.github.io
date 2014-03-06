---
author: admin
comments: true
date: 2012-11-07 12:54:23+00:00
layout: post
slug: pkg_resources-distributionnotfound
title: pkg_resources.DistributionNotFound
wordpress_id: 469
categories:
- 默认
tags:
- python
---

前段时间升级到Mountain Lion，之后很久没写过Python。昨天准备折腾一下才发现，我的Python竟然不工作了。python --version有输出，但pip报错：


    $ pip --version
    Traceback (most recent call last):
    File "/usr/local/bin/pip", line 5, in <module>
    from pkg_resources import load_entry_point
    File "/Library/Python/2.7/site-packages/distribute-0.6.30-py2.7.egg/pkg_resources.py", line 2807, in <module>
    working_set.require(__requires__)
    File "/Library/Python/2.7/site-packages/distribute-0.6.30-py2.7.egg/pkg_resources.py", line 690, in require
    needed = self.resolve(parse_requirements(requirements))
    File "/Library/Python/2.7/site-packages/distribute-0.6.30-py2.7.egg/pkg_resources.py", line 588, in resolve
    raise DistributionNotFound(req)
    pkg_resources.DistributionNotFound: pip==1.0.2


google了一晚上找到了解决方案

  * 在python的[setuptools页面](http://pypi.python.org/pypi/setuptools)下载对应版本的Python
  
  * 打开终端，运行：
    > `sh ./setuptools-0.6c11-py2.7.egg`

  * 完成之后运行：
    > `easy_install -U pip`

这个问题似乎是因为ML升级了Python，却没有做附带的升级site-packages工作。
