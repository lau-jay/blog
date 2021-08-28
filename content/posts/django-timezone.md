+++
title = "Django Timezone"
date = 2021-08-28T19:54:29+08:00
images = []
tags = ["timezone", "django"]
categories = ["Python"]
draft = false

+++

## 引言

线上遇到了个问题， 微信广告的订单回传，时间错了，然后去修这个时间错误。结果发现Django shell里执行datetime.utcnow的结果和now的结果以及time.time的结果居然是一模一样的。然后一路查找，先确认服务器的时间没问题，百思不解，意外在服务器上直接拉起原生的repl发现执行结果又正常了，然后去看代码发现Django的设置里时区被设置为utc了。那么上述所有的问题都得到解答了。所以下面记录下复习Django时区的过程, 如有不对欢迎指正。

## UTC和GMT+8

GMT：**G**reenwich **M**ean **T**ime 格林尼治标准时间。这是以英国格林尼治天文台观测结果得出的时间，这是英国格林尼治当地时间，这个地方的当地时间过去被当成世界标准的时间。

UT：**U**niversal **T**ime 世界时。根据原子钟计算出来的时间。

UTC：**C**oordinated **U**niversal **T**ime 协调世界时。因为地球自转越来越慢，每年都会比前一年多出零点几秒，每隔几年协调世界时组织都会给世界时+1秒，让基于原子钟的世界时和基于天文学（人类感知）的格林尼治标准时间相差不至于太大。并将得到的时间称为UTC，这是现在使用的世界标准时间。

协调世界时不与任何地区位置相关，也不代表此刻某地的时间，所以在说明某地时间时要加上时区

也就是说GMT并不等于UTC，而是等于UTC+0，只是格林尼治刚好在0时区上。

GMT = UTC+0

### **naive datetime object** vs **aware datetime object**

当使用datetime.now()得到一个datetime对象的时候，此时该datetime对象没有任何关于时区的信息，即datetime对象的tzinfo属性为None(tzinfo属性被用于存储datetime object关于时区的信息)，该datetime对象就被称为**naive datetime object**。

```python
>>> datetime.now().tzinfo
>>>
```

既然naive datetime object没有关于时区的信息存储，相对的aware datetime object就是指存储了时区信息的datetime object。在使用now函数的时候，可以指定时区，但该时区参数必须是datetime.tzinfo的子类

```python
>>> from django.utils.timezone import utc, is_aware
>>> import datetime
>>> datetime.datetime.now()
datetime.datetime(2021, 8, 28, 20, 30, 7, 922670)
>>> datetime.datetime.utcnow()
datetime.datetime(2021, 8, 28, 12, 30, 13, 591417)
>>> aware = datetime.datetime.now(utc)
>>> aware
datetime.datetime(2021, 8, 28, 12, 30, 46, 966581, tzinfo=<UTC>)
>>> is_aware(aware)
True
```

在Django中提供了几个简单的函数如is_aware, is_naive, make_aware和make_naive用于辨别和转换naive datetime object和aware datetime object。

