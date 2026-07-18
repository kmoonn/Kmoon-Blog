+++
title = "K 个点最小包围矩形"
date = "2026-07-18T19:40:04+08:00"
menuHidden = false
+++


## 问题描述

给定平面上 n 个二维坐标点，选出恰好 k 个点，用轴对齐矩形包围这 k 个点（矩形边平行坐标轴），求所有选法中矩形面积的最小值。

轴对齐矩形面积公式：

<img src="https://cdn.kmoon.fun/2026/2026-07-18T11-42-29-629Z.png" alt="" width=200/>

## 解法思路

考察点：滑动窗口

标准解法：

1. 所有点按 x 升序排序；
2. 枚举左边界 i（固定 x 最小的点）；
3. 不断扩大右边界 j（j 从 i 到 n-1），把 p [j] 的 y 加入临时 y 列表；
4. 对当前所有 y 排序，用滑动窗口取任意连续 k 个 y，得到最小 dy；
5. 当前矩形 dx = xj - xi，面积 dx*dy，更新全局最小值。

### 代码实现

```python
def minArea(points, k):
    n = len(points)
    if k == 1:
        return 0
    # 按x坐标升序排序所有点
    points.sort(key=lambda p: p[0])
    res = float("inf")

    for i in range(n):
        ys = []
        # 不断向右拓展右边界j
        for j in range(i, n):
            ys.append(points[j][1])
            ys.sort()
            # 滑动窗口找连续k个y的最小差值
            m = len(ys)
            # 只有当 m >= k 时，才会计算 dy 和面积
            for l in range(m - k + 1):
                dy = ys[l + k - 1] - ys[l]
                dx = points[j][0] - points[i][0]
                area = dx * dy
                if area < res:
                    res = area
    return res
```

## 剪枝优化

```python
# 剪枝1：过滤无法凑齐k个点的i
for i in range(n - k + 1):


# 剪枝2：当前dx已经大于最优，后续j只会更大，直接break内层
dx = points[j][0] - points[i][0]
if dx >= min_area:
    break
```

## 题目练习

- 209. 长度最小的子数组
- 302. 包含全部黑像素的最小矩形
- 939. 最小面积矩形
- 963. 最小面积矩形 II