---
title: Celery
date: 2017-07-10 16:11:29
categories: Python
tags: [Python, Celery]
---

## 终止进程
终止 Celery 相关进程：
``` bash
ps auxww | grep 'celery worker' | awk '{print $2}' | xargs kill -9
```

如果使用的是 celeryd：
``` bash
ps auxww | grep celeryd | awk '{print $2}' | xargs kill -9
```