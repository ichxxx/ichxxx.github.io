---
title: "Python代码技巧"
date: 2019-12-24T23:39:01+08:00
draft: false
tags: ["Python"]
---

### 交换值

```python
a, b = 5, 10
a, b = b, a
print(a,b)
# 10 5
```

### 连接列表所有元素为一个字符串

```python
a = ['Python', 'is', 'awesome']
print(' '.join(a))
# Python is awesome
```

### 寻找列表中频数最高的值

```python
a = [1,2,3,1,2,3,2,2,4,5,1]
f = max(set(a), key = a.count)
print(f)
# 2
```

### 判断两个字符串是否回文

```python
from collections import Counter
Counter(str1) == Counter(str2)
```

### 反转字符串/列表

```python
a = 'abcdefghijklmn'
b = [1, 2, 3, 4, 5]

print(a[::-1])
# nmlkjihgfedcba

print(b[::-1])
# [5, 4, 3, 2, 1]
```

### 转置二维数组

```python
a = [['a', 'b'], ['c', 'd'], ['e', 'f']]

a_t = zip(*a)
print(list(a_t))

# [('a', 'c', 'e'), ('b', 'd', 'f')]
```

### 链式比较

```python
a = 6

print(4 < a < 7)
# True

print(1 == a < 20)
# False
```

### 链式函数调用

```python
'''使用相同变量根据条件调用不同函数'''
def func_1(a, b):
  return a * b

def func_2(a, b):
  return a + b

print((func_1 if True else func_2)(5, 7))
# 35
```

### 复制列表

```python
a = [1, 2, 3, 4, 5]

'''这样子a和b的第一项都会改变'''
b = a
b[0] = 10

print(a, b)
# [10, 2, 3, 4, 5] [10, 2, 3, 4, 5]

'''这样子只有b的第一项会改变'''
b = a[:]
b[0] = 10

print(a, b)
# [1, 2, 3, 4, 5] [10, 2, 3, 4, 5]

'''使用类型转换复制列表'''
b = list(a)

'''使用 list.copy() 方法复制列表（只支持Python3）'''
b = a.copy()

'''使用 copy.deepcopy 复制嵌套列表'''
from copy import deepcopy

l = [[1, 2], [3, 4]]
l2 = deepcopy(l)
```

### 获取字典值

```python
'''当键值不在字典里时，返回 None 或者默认值'''
d = {'a': 1, 'b': 2}

print(d.get('c'))
# None

print(d.get('c', 3)
# 3
```

### 按字典的值排序

```python
d = {'a': 10, 'b': 20, 'c': 5, 'd': 1}

print(sorted(d.items(), key = lambda x: x[1]))
# [('d', 1), ('c', 5), ('a', 10), ('b', 20)]

print(sorted(d, key=d.get))
# ['d', 'c', 'a', 'b']
```

### For Else

```python
'''当for循环没有触发嵌套的if判断时，会触发else'''
a = [1, 2, 3, 4, 5]

for each in a:
  if each == 0:
    print('bad')
else:
  print('awesome')

# awesome
```

### 将列表转换为符号分割的字符串

```python
items = ['foo', 'bar', 'xyz']
print(','.join(items))
# foo,bar,xyz

'''数字列表转换'''
nums = [1, 2, 3, 4]
print(','.join(map(str, nums)))
# 1,2,3,4

'''混合列表转换'''
data = [1, 'Hello', 2, 3.4]
print(','.join(map(str, nums)))
# 1,Hello,2,3.4
```

### 合并字典

```python
d1 = {'a': 1}
d2 = {'b': 2}

# python3.5
print({**d1, **d2})
# {'a': 1, 'b': 2}

print(dict(d1.items() | d2.items()))
# {'b': 2, 'a': 1}

d1.update(d2)
print(d1)
# {'a': 1, 'b': 2}
```

### 列表中的最大和最小索引

```python
l = [20, 40, 10, 30]

print(l.index(min(l)))
# 2

print(l.index(max(l)))
# 1
```

### 删除列表中的重复项

```python
'''删除后不保持原列表的顺序'''
items = [2, 2, 3, 3, 1]

print(list(set(items)))
# [1, 2, 3]

'''删除后保持原列表顺序'''
from collections import OrderedDict

print(list(OrderedDict.fromkeys(items).keys()))
# [2, 3, 1]
```

翻译和修改自 《[Python Tricks 101](https://hackernoon.com/python-tricks-101-2836251922e0)》 by Gautham Santhosh