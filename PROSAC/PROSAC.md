https://blog.csdn.net/Rotating_Sky/article/details/87929195

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

# 参数定义

共有 N 个数据点（特征点的总量为 N）；根据评价函数 q （我猜是汉明距离……）；前 n 名记为数据集 $U_n$；采样点集合记为 $M \subset U_n$ （M 的大小为 4）；我们认为，用 RANSAC 算法，经过 $T_N$ 次采样之后，RANSAC 算法与 PROSAC 算法的效果一致。

显然， $T_N$ 和 $T_n$ 的数量关系满足 $ T_n=T_N \frac{ \begin{pmatrix} n \\ m \end{pmatrix} }{ \begin{pmatrix} N \\ m \end{pmatrix} } $；由此可以推出 $\Rightarrow T_{n+1}=\frac{n+1}{n+1-m} T_n $；由此我们定义 $T'_{n+1}=T'_n + \lceil T_{n+1}-T_n \rceil$。当 n 更新后，$ T'_n $ 随之更新。

e.g. 取 $T_N = \begin{pmatrix} N/5 \\ m \end{pmatrix} $，就可以控制 $T'_{n+1} - T'_n$ 不会取遍所有的 $u_{n+1}$ 点。而如果 $T_N = \begin{pmatrix} N \\ m \end{pmatrix} $，则会取遍所有的 $u_{n+1}$ 点。

# 终止策略

## 非随机性

假设某次采样计算得到了一个错误模型，错误模型在数据集 N 中有若干个类内点。设数据集 N 中的一个点是错误模型类内点的概率为 $\beta$，则 $U_n$ 中有 i 个类内点的概率为 $P_n^R(i) = \beta ^{i-m} (1-\beta)^{n-i+m} \begin{pmatrix} n-m \\ i-m \end{pmatrix}$。

通过预设 $\Psi$ 值为 5%，可以计算得出最少类内点数 $I_{n'^*}^{min} = min \{ j: \sum_{i=j}^{n'^*} {P_{n'^*}^R(i) < \Psi} \}$ ，（本例中 $ n'^* $ 代表类内点的最大序号）。当某一模型类内点数大于这一最少数值时，我们就有足够把握用 $n'^*$ 更新 $n^*$。（$n^*$ 是数据集 $U_n$ 大小的上限）

## 极大性

极大性是 RANSAC 唯一的终止策略，根据 $n^*$ 和当前模型的类内点数量，更新迭代次数。

# 算法流程

$t:=0, n:=m, n^*:=N, T'_n:=1$

循环开始，直至达到迭代次数。

## Step1

$t:=t+1$

if ($t=T'_n$) & ($ n < n^* $) then ($n:=n+1$; update $T'_n$)

## Step2

if ($T'_n > t$) then The sample contains $m-1$ points selected from $U_{n-1}$ at random and $u_n$

（论文中是 <，但我觉得不对)

else select m points from $U_n$ at random

## Step3

Compute model parameters $p_t$ from the sample $M_t$

## Step4

Find support (i.e. consisitent data points) of the model with parameters $p_t$

Select termination length $n^*$ if possibl. 

并用更新后的 $n^*$ 更新迭代次数；根据重投影误差，对特征点重新打分并排序。



