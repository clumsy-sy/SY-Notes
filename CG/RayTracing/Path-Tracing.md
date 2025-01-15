# Path Tracing

## 定义

路径追踪（PT）是一种基于光线追踪的算法，通过模拟光线在场景中的传播和相互作用来计算图像上每个像素的颜色。

## 基础知识

### 渲染方程（The Rendering Equation）

$$
L_{out}(p, \omega_{out}) = L_{emit}(p, \omega_{out}) + \int_{\Omega^+} f_{BRDF}(p, \omega_{in}, \omega_{out}) L_{in}(p, \omega_{in})(n \cdot \omega_{in}) \mathbf{d}\omega_{in}
$$

简单的说，从 $\omega_{out}$ 方向观测 $p$ 点的 radiance，由两部分决定
1. $p$ 点自身发射的 radiance。
2. $p$ 点吸收来自各个方向的 irradiance（也就是积分），根据该点的材质不同，有不同的 BRDF ，决定了发射向观测方向的 radiance。

### 蒙特卡洛积分（Monte Carlo Integration）

渲染方程中含有一个定积分，这个积分比较难直接求解，一种比较适合计算机求解的方法，便是蒙特卡洛积分。

#### 公式

假设现在要对一个函数 $f(x)$ 进行定积分操作，也就是

$$
\int_a^b f(x) \mathbf{d}x
$$

找一个**合适** 的随机样本，概率密度函数为 $p(x)$ 取得随机遍历 $X_i$

$$
X_i \sim p(x)
$$

可以得到蒙特卡洛估计量 $F_N$

$$
F_N = \dfrac{1}{N} \sum_{i = 1}^{N} \dfrac{f(X_i)}{p(X_i)} 
$$

以均匀分布为例

$$
\begin{align}
\int_a^b &f(x) \mathbf{d}x \\
X_i &\sim p(x) = \dfrac{1}{b - a}\\
F_N &= \dfrac{b - a}{N} \sum_{i = 1}^{N}f(X_i)
\end{align}
$$

#### 重要性采样

$$
F_N = \dfrac{1}{N} \sum_{i = 1}^{N} \dfrac{f(X_i)}{p(X_i)} , X_i \sim p(x)
$$

从公式可以看出任意的概率分布函数 $p(x)$ 都可以通过蒙特卡洛积分得到正确的值，但是在取样数固定的情况下是否有更好的 $p(x)$，在相同的样本量，比原始分布采样得到的结果方差小。

我目前的理解是，重要性采样是通过特定的 $pdf$ 采样，从而在容易产生 noise 的地方多采样，也可以认为是将采样引向原分布中更重要的部分（例如光源，反射），当然更重要的部分可能在数值上也会比周边大。


## 算法基本思想

### 伪代码

```cpp
auto shade(p，w_out) -> color
  // Contribution from the light source. 
  Uniformly sample the light at x’ (pdf_light = 1 / A)
  Shoot a ray from p to x’
  If the ray is not blocked in the middle
    L_dir = L_in * f_brdf * cos θ * cos θ’ / |x’ - p|^2 / pdf_light 

  // Contribution from other reflectors.
  L_indir = 0.0
  Test Russian Roulette with probability P_RR

  Uniformly sample the hemisphere toward wi (pdf_hemi = 1 / 2pi)
  Trace a ray r(p, w_in)
  If ray r hit a non-emitting object at q
  L_indir = shade(q, -w_in) * f_brdf * cos θ / pdf_hemi / P_RR

  return L_dir + L_indir
```

## 实现

[简单实现](https://github.com/clumsy-sy/Ray-Tracing)

## 来源

[Ray Tracing in One Weekend](https://raytracing.github.io/)
[games101 ray tracing](https://www.bilibili.com/video/BV1X7411F744?p=13&vd_source=3825e75351cb6b995f61e3725b76ff79)


