---
title:      " 一行 Python 代码能实现这么多丧心病狂的功能？"
date:       2019-12-13
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Python ]

---

# 一行代码打印乘法口诀

```python
print('\n'.join([' '.join(["%2s x%2s = %2s"%(j,i,i*j) for j in range(1,i+1)]) for i in range(1,10)]))
```

# 一行代码打印迷宫

```python
print(''.join(__import__('random').choice('\u2571\u2572') for i in range(50*24)))
```

# 一行代码表白爱情

```python
print('\n'.join([''.join([('Love'[(x-y) % len('Love')] if ((x*0.05)**2+(y*0.1)**2-1)**3-(x*0.05)**2*(y*0.1)**3 <= 0else' ') for x in range(-30, 30)]) for y in range(30, -30, -1)]))！
```

# 一行代码打印小龟龟

```python
print('\n'.join([''.join(['*' if abs((lambda a:lambda z,c,n:a(a,z,c,n))(lambda s,z,c,n:z if n==0 else s(s,z*z+c,c,n-1))(0,0.02*x+0.05j*y,40))<2 else ' ' for x in range(-80,20)]) for y in range(-20,20)]))
```

# 一行代码实现 1 - 100 的和

```python
sum(range(1,101))
```

# 一行代码实现数值交换

```python
a = 1
b = 2
a,b = b,a
print(a,b)
```

# 一行代码求奇偶数

```python
[x for  x in range(10) if x % 2 == 1]
```

# 一行代码展开列表

```python
list = [[1,2,3],[4,5,6],[7,8,9]]
[j for i in list for j in i]
```

# 一行代码打乱列表

```python
import random
lst = [1,2,3,4,5]
random.shuffle(lst)
lst
```

# 一行代码反转字符串

```python
name = 'byf4963cg'
name = [::-1]
```

# 一行代码查看目录下所有文件

```python
import os
os.listdir('.')
```

# 一行代码去除字符串间的空格

```python
des = 'My name is master123'
des.replace("","")
```
```python
des = 'My name is master123'
"".join(des.split(" "))
```

# 一行代码实现字符串整数列表变成整数列表

```python
a = ['1','2','3','4','5']
list(map(lambda a : int (a),['1','2','3','4','5']))
```

# 一行代码删除列表中重复的值

```python
lst = [1,1,3,3,5]
list(set(lst))
```

# 一行代码找出两个列表中相同的元素

```python
a = [1,1,2,2,4]
b = [2,3,4,4]
set(a) & set(b)
```

# 一行代码找出两个列表中不同的元素

```python
a = [1,1,2,2,4]
b = [2,3,4,4]
set(a) ^ set(b)
```

# 一行代码合并两个字典

```python
des = {'name': 'john'}
age = {'age': '25'}
des.update(age)
des
```

# 一行代码实现字典键从小到大排序

```python
des = {'name': 'john','age': '25','like': 'python'}
sorted(des.items(), key=lambda x: x[0])
```