---
layout: post
title: Python学习 - MATLAB与numpy矩阵操作对比
author: 宋强
tags: Python numpy
---

# 创建矩阵
Matlab
```matlab
m1 = [0, 3, 1;
      1,-1, 1;
      3,-1, 2]
m2 = [1 2 3;
      4 5 6;
      7 8 9]
m3 = [-1.1 1.5 1.3]
```
Python
```python
m1 = mat([[0, 3, 1],
          [1,-1, 1],
          [3,-1, 2]])
m2 = mat([[1, 2, 3],
          [4, 5, 6],
          [7, 8, 9]])
m3 = mat([-1.1, 1.5, 1.3]) 
print(m1)
print(m2)
print(m3)
```

# 矩阵转置
Matlab
```matlab
mT = m1'
```
Python
```Python
mT = m1.T
mT = m1.transpose()
```

# 矩阵求逆
Matlab
```matlab
mI = inv(m1)
```
Python
```python
mI = m1.I
```

# 矩阵元素级(element-wise)相加
Matlab
```matlab
mSum = m1 + m2
```
Python
```python
mSum = add(m1, m2)
```

# 矩阵元素级相减
Matlab
```matlab
mSubtract = m1 - m2
```
Python
```python
mSubtract = subtract(m1, m2)
```

# 矩阵元素级相乘
Matlab
```matlab
mMultiply = m1.*m2
```
Python
```python
mMultiply = np.multiply(m1, m2)
```

# 矩阵元素级相除
Matlab
```matlab
mDivide = m1./m2
```
Python
```python
mDivide = divide(m1, m2)
```

# 矩阵元素级取反
Matlab
```matlab
mNegtive = -m1
```
Python
```python
mNegtive = -m1
mNegtive = np.negative(m1)
```

# 矩阵元素级幂
Matlab
```matlab
mPower = m1.^3
```
```python
mPower = np.power(m1, 3)
```
