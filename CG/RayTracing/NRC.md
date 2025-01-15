# Real-time Neural Radiance Caching for Path Tracing

## 1. 导言

路径追踪全局光照是实时渲染中的难题,辐射量具有显著的空间、方向和时间相关性，可以通过各种方式利用这些相关性来加速渲染。

系统设计的三个原则：
1.  动态内容。系统必须支持摄像机、照明、几何体和材料的任意动态。
2.  鲁棒性。缓存应与材料和场景几何无关。这一点对于用户生成的内容尤为重要，因为用户生成的内容无法轻易执行设计假设。不过，可以向系统提供额外的属性，以提高渲染质量。
3.  可预测的性能和资源消耗。工作负载和内存使用量的波动会导致帧频不稳定。渲染成本必须与像素数量成最坏的线性比例。

### 2. 相关工作

1. Radiance caching
2. Precomputation-based techniques
3. Fully dynamic techniques
4. Path guiding
5. Neural techniques


## 3.预备知识

### PT

### IS MIS NEE ReSTIR

### MLP

## 理论

NRC 目标是使用一个神经网络来缓存辐射度（radiance），该网络可将空间方向坐标（spatio-directional coordinates）映射为辐射度值（radiance value）

$$
L_{out}(p, \omega_{out}) = L_{emit}(p, \omega_{out}) + \int_{\Omega^+} f_{BRDF}(p, \omega_{in}, \omega_{out}) L_{in}(p, \omega_{in})(n \cdot \omega_{in}) \mathbf{d}\omega_{in}
$$

通过神经网络，得到 $\widehat{L}$ 来近似 $L_{out}(p, \omega_{out})$.

**整个渲染过程分为两步**

1. 追踪较短的渲染路径，1pps，一旦辐照度缓存提供的近似值被认为足够精确，终止路径。也是只有当路径传播的范围够大，足以模糊缓存的误差的时候才会直接查询缓存。在每个中间顶点，使用 NEE 来整合来自发射器的光线。在主顶点使用屏幕空间 ReSTIR ，在后续顶点使用 LightBVH，通过 MIS 结合 BRDF 采样。在 $x_k$ 处中断，评估 $L_{out}(p, \omega_{out})$ 与 NRC 给出的值是否接近.

$$
a(x_1 \cdots x_n) = \left ( \sum_{i=2}^{n} \sqrt{\dfrac{||x_{i-1}-x_i||^2}{p(\omega_i |x_{i-1},\omega)|\cos \theta_i|)}} \right ) ^2
$$

$$
a_0:=\dfrac{||x_0-x_1||^2}{4\pi\cos\theta_1}
$$

当 $a(x_1··x_n) > c · a_0$ 时终止路径，c是一个超参数，它在偏差(较长路径)和速度(较短路径)之间进行权衡。$c = 0.01$ 得到了令人满意的结果, 如果路径被选择成为训练路径，则将再次使用启发式，这一次是在满足 $a(x_n···x_m)>c·a_0$时终止训练后缀。



2. 为了训练辐照度缓存，我们将短渲染路径的一部分（通常低于 3%）延长几个顶点--这就是训练后缀（training suffix）。与之前一样，一旦后缀的面积分布足够大，我们就会终止。在大多数情况下，后缀由一个顶点组成。沿着较长训练路径的所有顶点收集的辐射度估计值将用作训练辐射度缓存的参考值(用于更新网络)。每第16个训练后缀只以俄罗斯轮盘赌结束。


缺点：
1. 训练路径的最后一个顶点可能会到达辐射缓存尚未训练过的场景位置，这可能会产生较大的近似误差。
2. 迭代优化可能只模拟多重反弹照明的一个子集，而不是所有的光传输。


训练：

当渲染动态内容时，例如改变摄像机位置或动画几何形状，神经辐射缓存需要不断地适应，迫使我们在通过梯度下降来优化网络时使用高学习率，但这会造成时间伪像（如在所绘制的帧上的闪烁和振荡）。



可能的伪代码，没有加 ReSTIR
```cpp

NetWork MLP;

auto mlp_forward()
auto mlp_backward()
auto mlp_train( training suffix ... )
auto mlp_predict( omega )

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
    If miss small enought
      // cache from mlp
      mlp_predict() * f_brdf * cos θ / pdf_hemi / P_RR
      if train
        train suffix = shade(q, -w_in)
        // update mlp
        mlp_train(train suffix)
    Else 
      L_indir = shade(q，r.dir) * f_brdf * cos θ / pdf_hemi / P_RR

  return L_dir + L_indir
```