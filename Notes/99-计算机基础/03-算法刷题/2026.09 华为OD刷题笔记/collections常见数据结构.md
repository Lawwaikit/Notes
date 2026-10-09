# 1. `Counter`：计数器

`Counter` 本质上是一个“自动计数的字典”，特别适合解决**统计频率**的问题。

```python
#例如统计数组中每个数字出现了几次：
from collections import Counter

nums = [1, 2, 2, 3, 3, 3]

count = Counter(nums)

print(count)
# Counter({3: 3, 2: 2, 1: 1})

print(count[3])
# 3
```

Counter 额外支持字典中没有的三个方法：`elements()`, `most_common([m])`, `subtract([iterable-or-mapping])`。

`most_common([n])` 方法是 Counter 最常用的方法，返回一个出现次数从大到小的前 n 个元素的列表。

```python

from collections import Counter

c = Counter({'a':1, 'b':2, 'c':3})

>>> print(c.most_common()) # 默认参数
[('c', 3), ('b', 2), ('a', 1)]
>>> print(c.most_common(2)) # n = 2
 [('c', 3), ('b', 2)] 
>>> print(c.most_common(3)) # n = 3
[('c', 3), ('b', 2), ('a', 1)] 
>>> print(c.most_common(-1)) # n = -1
[]
```

`subtract([iterable_or_mapping])`方法其实就是将两个 Counter 对象中的元素对应的计数相减。

```python
from collections import Counter

c = Counter({'a':1, 'b':2, 'c':3})
d = Counter({'a':1, 'b':3, 'c':2, 'd':2})
c.subtract(d)

>>> print(c)
Counter({'c': 1, 'a': 0, 'b': -1, 'd': -2})
```


# 2. `defaultdict`：带默认值的字典

普通 `dict` 有一个比较麻烦的地方：

```python
d = {}

d["a"].append(1)
```

会报错，因为 `"a"` 还不存在。

通常需要：

```python
d = {}

if "a" not in d:
    d["a"] = []

d["a"].append(1)
```

`defaultdict` 可以自动创建默认值：

```python
from collections import defaultdict

d = defaultdict(list)

d["a"].append(1)
d["a"].append(2)
d["b"].append(3)

print(d)
# defaultdict(<class 'list'>, {'a': [1, 2], 'b': [3]})
```

这里：

```python
defaultdict(list)
```

意思是：如果访问一个不存在的 key，就自动创建一个空 `list`。所以：

```python
d["a"].append(1)

# 第一次访问 `"a"` 时，相当于自动做了：
d["a"] = []
#然后
d["a"].append(1)
```


### 用途一：分组

例如：

```python
nums = [1, 2, 3, 4, 5, 6]

d = defaultdict(list)

for x in nums:
    d[x % 2].append(x)

print(d)
```

结果相当于：

```python
{
    0: [2, 4, 6],
    1: [1, 3, 5]
}
```

也就是：**按照某个条件，把数据分组。**

### 用途二：构建图

这是算法题里非常重要的用法。

例如：

```python
edges = [
    (1, 2),
    (1, 3),
    (2, 4)
]

from collections import defaultdict

graph = defaultdict(list)

for u, v in edges:
    graph[u].append(v)

#得到：graph=
{
    1: [2, 3],
    2: [4]
}
```

之后 DFS / BFS 就非常方便：

```python
for next_node in graph[node]:
    ...
```

所以以后看到：

> 给你一些边，构建邻接表

基本可以想到defaultdict


### 用途三：统计字典

其实 `defaultdict(int)` 也非常常用：

```python
from collections import defaultdict

count = defaultdict(int)

for x in nums:
    count[x] += 1
```

不过如果只是单纯计数，通常直接：

```python
Counter(nums)
```

更加简洁。


# 4. `deque`：双端队列

它可以在**两端快速插入和删除**。

```python

from collections import deque
q = deque()

#
q.append(1)
q.append(2)
q.append(3)

print(q)
# deque([1, 2, 3])

# 右边操作
q.append(4)
q.pop()

# 左边操作
q.appendleft(0)
q.popleft()
```

例如：

```python
q = deque([1, 2, 3])

q.appendleft(0)
print(q)
# deque([0, 1, 2, 3])

q.popleft()
print(q)
# deque([1, 2, 3])
```


### 用途一：BFS 广度优先搜索

例如二叉树层序遍历：

```python
from collections import deque

q = deque([root])

while q:
    node = q.popleft()

    if node.left:
        q.append(node.left)

    if node.right:
        q.append(node.right)
```

这也是你做 BFS 时应该形成的肌肉记忆。


### 其余用途：队列，栈，单调队列，滑动窗口

它不只是队列。
因为两边都能操作，所以可以用来实现：

**队列：**

```
q.append(x)
q.popleft()
```

**栈：**

```
stack.append(x)
stack.pop()
```

**双端队列：**

```
q.append(x)
q.appendleft(x)

q.pop()
q.popleft()
```

尤其是在一些**滑动窗口、单调队列**问题中非常重要。

例如经典的“滑动窗口最大值”，通常就会使用：deque维护一个单调队列。


# 7. `OrderedDict`：有序字典

```python
from collections import OrderedDict
```

例如：

```python
d = OrderedDict()

d["a"] = 1
d["b"] = 2
d["c"] = 3

print(d)
# {'a': 1, 'b': 2, 'c': 3}
```

### 特殊方法
`OrderedDict` 还有一些特殊操作，例如：

```python
popitem(last=True)

move_to_end(last=True)
# 如果last为True（默认值），则移动到末尾
# 如果last为False，则移动到开头
# 如果key不存在，引发KeyError

```

### OrderedDict与sort结合

由于有序字典会记住其插入顺序，因此可以与排序结合使用以创建排序字典：

```python
>>> # 标准未排序的常规字典
>>> d = {'banana': 3, 'apple': 4, 'pear': 1, 'orange': 2}

>>> # 按照key排序的字典
>>> OrderedDict(sorted(d.items(), key=lambda t: t[0]))
OrderedDict([('apple', 4), ('banana', 3), ('orange', 2), ('pear', 1)])

>>> # 按照value排序的字典
>>> OrderedDict(sorted(d.items(), key=lambda t: t[1]))
OrderedDict([('pear', 1), ('orange', 2), ('banana', 3), ('apple', 4)])

>>> # 按照key的长度排序的字典
>>> OrderedDict(sorted(d.items(), key=lambda t: len(t[0])))
OrderedDict([('pear', 1), ('apple', 4), ('orange', 2), ('banana', 3)])
```