---
title:      " 六种酷炫Python运行进度条 "
date:       2020-09-29 16:37
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Python ]
---

## 1.普通进度条

在代码迭代运行中可以自己进行统计计算，并使用格式化字符串输出代码运行进度

```
import sys
import time
def progress_bar():
    for i in range(1, 101):
        print("\r", end="")
        print("Download progress: {}%: ".format(i), "▋" * (i // 2), end="")
        sys.stdout.flush()
        time.sleep(0.05)
progress_bar()
```



## 2.带时间进度条

导入time模块来计算代码运行的时间，加上代码迭代进度使用格式化字符串来输出代码运行进度

```
import time
scale = 50
print("执行开始，祈祷不报错".center(scale // 2,"-"))
start = time.perf_counter()
for i in range(scale + 1):
    a = "*" * i
    b = "." * (scale - i)
    c = (i / scale) * 100
    dur = time.perf_counter() - start
    print("\r{:^3.0f}%[{}->{}]{:.2f}s".format(c,a,b,dur),end = "")
    time.sleep(0.1)
print("\n"+"执行结束，万幸".center(scale // 2,"-"))
```



## 3.tpdm进度条

这是一个专门生成进度条的工具包，可以使用pip在终端进行下载，当然还能切换进度条风格

```
from time import sleep
from tqdm import tqdm
# 这里同样的，tqdm就是这个进度条最常用的一个方法
# 里面存一个可迭代对象
for i in tqdm(range(1, 500)):
   # 模拟你的任务
   sleep(0.01)
sleep(0.5)
```



## 4.progress进度条

你只需要定义迭代的次数、进度条类型并在每次迭代时告知进度条即可，具体代码案例如下

```
import time
from progress.bar import IncrementalBar
mylist = [1,2,3,4,5,6,7,8]
bar = IncrementalBar('Countdown', max = len(mylist))
for item in mylist:
    bar.next()
    time.sleep(1)
    bar.finish()
```




## 5.alive_progress进度条

顾名思义，这个库可以使得进度条变得生动起来，它比原来我们见过的进度条多了一些动画效果，需要使用pip进行下载，代码案例如下：

```
from alive_progress import alive_bar
items = range(100)                  # retrieve your set of items
with alive_bar(len(items)) as bar:   # declare your expected total
    for item in items:               # iterate as usual
        # process each item
        bar()
        time.sleep(0.1)
```



## 6.可视化进度条

用 PySimpleGUI 得到图形化进度条，我们可以加一行简单的代码，在命令行脚本中得到图形化进度条，也是使用pip进行下载，代码案例如下

```
import PySimpleGUI as sg
import time
mylist = [1,2,3,4,5,6,7,8]
for i, item in enumerate(mylist):
    sg.one_line_progress_meter('This is my progress meter!', i+1, len(mylist), '-key-')
    time.sleep(1)
```

