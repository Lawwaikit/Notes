并查集（Disjoint Set Union，DSU）主要解决一类问题：**动态维护“哪些元素属于同一个集合”**。它最核心的操作只有两个：`find(x)` 查询 `x` 属于哪个集合，`union(a, b)` 把 `a` 和 `b` 所在的两个集合合并。

例如有 `1, 2, 3, 4, 5` 五个人，最开始每个人都是独立集合。现在知道 `1` 和 `2` 是朋友，`2` 和 `3` 是朋友，那么可以合并成 `{1,2,3}`；之后查询 `1` 和 `3` 是否属于同一个集合，只需要判断它们的“代表节点”是否相同。

```python
# 并查集，每个区域起始代表为自己

# parent 表示父节点
parents = []

# find 找根节点并进行路径压缩
def get_parent(x):
    if parents[x] != x:
        parents[x] = get_parent(parents[x])
    return parents[x]

# union 合并两个根节点
def union(x, y):
    xp = get_parent(x)
    yp = get_parent(y)

    m = min(xp, yp)
    parents[xp] = m
    parents[yp] = m
```