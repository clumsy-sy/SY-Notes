# Sampling

## IS (Importance Sampling)

$$
F_N = \dfrac{1}{N} \sum_{i = 1}^{N} \dfrac{f(X_i)}{p(X_i)} , X_i \sim p(x)
$$

从公式可以看出任意的概率分布函数 $p(x)$ 都可以通过蒙特卡洛积分得到正确的值，但是在取样数固定的情况下是否有更好的 $p(x)$，在相同的样本量，比原始分布采样得到的结果方差小。

什么时候方差最小？就是 $\dfrac{f(X_i)}{p(X_i)}$ 接近于常值函数的时候，也就是 $f(x)$ 和 $p(x)$ 的形状尽可能相似的时候。

值得注意的一点，$p(x)$ 的域必须覆盖 $f(x)$ 的整个域，否则不成立。如果 $\Omega \notin p(x)$ 但是 $f(x)$ 在 $\Omega$ 内的值为 $0$ 则也可行。

我目前的理解是，重要性采样是通过特定的 $pdf$ 采样，从而在容易产生 noise 的地方多采样，也可以认为是将采样引向原分布中更重要的部分（例如光源，反射），当然更重要的部分可能在数值上也会比周边大。

## MIS (Multiple Important Sampling)

$$
I_{mis}^{M,N} = \sum_{i=1}^M \dfrac{1}{N_i} \sum_{j=1}^{N_i} w_i(x_{ij}) \dfrac{f (x_{ij})}{p_i(x_{ij})} , \sum_{i=1}^M w_i(x) = 1
$$

其实与重要性采样相比就是多了 $\omega_i(x)$，这样就可以把不同的采样策略结合起来。最简单的便是均匀权重 $\omega_i(x) = \dfrac{1}{M}$，用得最多的平衡启发式（balanced heuristics）$w_i(x)=\dfrac{N_i p_i(x)}{\sum_{k=0}^M N_kp_k(x)}$ 这样可以实现类似复杂的多峰 $pdf$

## NEE Next Event Estimation

分离直接光照与间接光照，两种实现
1. 直接把除第一次外的亮度 / 2 （感觉有问题）
2. 只保留第一次自发光，只有在直接光照采样的部分打到光源，才返回光源的 emission，如果在间接光照采样的部分打到光源，返回 0，镜面等需要特殊处理

## SIR Sample Importance Resampling

提议分布（Proposal PDF）$p(x)$可以是很简单的一个分布，比如均匀分布
目标分布（Target PDF）$\hat p(x)$ 是我们可以进行重要性采样的分布

流程：
1. 从提议分布中取 $M$ 个样本
2. 对每个样本分配一个权重 $\omega(x) = \dfrac{\hat p(x)}{p(x)}$
3. 接下来从这个 $M$ 个样本中抽取 $n$ 个样本。每个样本被抽取的概率与其权重成正比, 这也会服从一种分布，称为 SIR PDF 或者 real PDF， 随着 $n$ 的增大， SIR PDF 会越来越接近目标分布

它为我们提供了一个强大的工具，可以从我们无法用其他方法采样的分布中抽取样本，因为实际上目标分布可以是分布函数而非 PDF。

## RIS (Resampled Importance Sampling)

RIS 使用的便是上述 $n$ 个 SIR 样本
$$
I_{RIS} = \dfrac{1}{N} \Sigma_{i=1}^{N} \dfrac{f(x_i)}{\hat{p}(x_i)} \Sigma_{j=1}^ {M} w(x_j)
$$

重新完整描述一下

假设我们有一个极度贴近实际分布的好分布 $\hat p$，但它不好生成样本，只适合评估样本；另一方面，我们又有另一个比较简单的分布 $p$，虽然它简单地足以能够生成样本，但是这个分布又比较偏离实际分布。

RIS：先用简单分布 $p$ 生成 $M$ 个样本，再从这 $M$ 个样本用复杂分布 $\hat p$ 再挑出 1 个样本(当然实际上挑样本是根据权重来挑选的)（要用 RIS 输出 $N$ 个样本，也只需要独立地调用 $N$ 次 one-sample RIS 即可）。RIS 的效果一般来说要好过纯粹用 $p$ 来进行 IS。

## WRS Weighted Reservoir Sampling

考虑上面的，提议分布中 $M$ 越大越好，但这会用很多存储空间，WRS 聪明地将 RIS 的算法改造成了流式算法，让空间复杂度与 $M$ 无关（只需要记录三个属性），这就意味着甚至 $M$ 无穷大都可以容纳（精度足够的情况下）。

核心就是改造成了一个蓄水池结构（reservoir），它只需要存储：

$y$ ：一个用于输出的样本，也可以理解成经过 $M$ 次输入后留下来的 winner 样本。\
$w_{sum}$：输入过的样本的权重 $w$ 总和。\
$M$：输入过的样本数量。

然后可以源源不断给蓄水池输入新样本，每次输入新样本时它便会根据权重 $w$ 去决定留下旧样本还是留下新样本。

## ReSTIR

基于储层的时空重要性采样：
时空复用 + WRS with MIS

ReSTIR DI : 对大量光源的采样，时间重用，相邻像素，M 重用 使用 WRS + MIS


## 参考资料

[lightBVH](https://www.cnblogs.com/KillerAery/p/17202961.html#importance-samplingis)

[ReSTIR DI 数学理解1](https://zhuanlan.zhihu.com/p/670309912)

[ReSTIR DI 数学理解2](https://zhuanlan.zhihu.com/p/591979133)

[ReSTIR 论文](https://research.nvidia.com/publication/2020-07_spatiotemporal-reservoir-resampling-real-time-ray-tracing-dynamic-direct)
