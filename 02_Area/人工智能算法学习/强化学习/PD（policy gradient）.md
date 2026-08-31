

策略 $\pi_\theta$ 是一个带参数 $\theta$ 的策略网络（比如神经网络权重）。核心问题是：

如何调整 $\theta$，让智能体从环境中获得的**期望回报**越来越大？

策略梯度方法就是直接对这个目标求梯度，然后做梯度上升。

## 第一步：定义优化目标

先明确“好”是什么意思。定义目标函数为：从初始状态出发，遵循策略 $\pi_\theta$，期望能够拿到多少总回报：

$$
J(\theta) = \mathbb{E}_{\tau \sim \pi_\theta}[G(\tau)]
$$

这里：

- $\tau = (s_0, a_0, r_1, s_1, a_1, r_2, \dots)$ 表示一整条轨迹（trajectory）
- $G(\tau)$ 表示这条轨迹的总折扣回报

我们的目标就是做梯度上升：

$$
\theta \leftarrow \theta + \alpha \nabla_\theta J(\theta)
$$

问题在于：$J(\theta)$ 是对轨迹分布的期望，而轨迹分布本身又依赖于 $\theta$。也就是说，策略变了，采样到的轨迹分布也变了。  
那么，怎么对“依赖参数的期望”求梯度，并且让这个梯度还能通过采样来估计？这正是策略梯度定理要解决的核心问题。

## 第二步：log-derivative trick（对数导数技巧）

这是整个推导的灵魂。先用一个纯数学恒等式：

$$
\nabla_\theta \pi_\theta(x)
= \pi_\theta(x) \cdot \frac{\nabla_\theta \pi_\theta(x)}{\pi_\theta(x)}
= \pi_\theta(x) \cdot \nabla_\theta \log \pi_\theta(x)
$$

本质上只是利用了链式法则：

$$
\nabla_\theta \log f(\theta) = \frac{\nabla_\theta f(\theta)}{f(\theta)}
$$

把目标函数按定义展开：

$$
J(\theta) = \sum_\tau P_\theta(\tau) G(\tau)
$$

对 $\theta$ 求梯度：

$$
\nabla_\theta J(\theta)
= \sum_\tau \nabla_\theta P_\theta(\tau) \cdot G(\tau)
$$

直接对 $P_\theta(\tau)$ 求梯度并不好处理，因为这是一个复杂分布，不方便直接采样其梯度。  
这时套用上面的技巧：

$$
\nabla_\theta P_\theta(\tau)
= P_\theta(\tau) \cdot \nabla_\theta \log P_\theta(\tau)
$$

代回去得到：

$$
\nabla_\theta J(\theta)
= \sum_\tau P_\theta(\tau)\,\nabla_\theta \log P_\theta(\tau)\,G(\tau)
= \mathbb{E}_{\tau \sim \pi_\theta}
\left[\nabla_\theta \log P_\theta(\tau) \cdot G(\tau)\right]
$$

这一步非常关键：  
原来“不好对付的期望梯度”，被改写成了“一个可以通过采样估计的期望”。  
这也是策略梯度方法能够成立的关键转折点。

## 第三步：把 $\log P_\theta(\tau)$ 拆开，环境部分自动消失

一条轨迹的概率由两部分组成：

1. 策略在每个状态下选动作的概率
2. 环境的状态转移概率

因此：

$$
P_\theta(\tau)
= \rho_0(s_0)\prod_{t=0}^{T-1}\pi_\theta(a_t \mid s_t)\,P(s_{t+1} \mid s_t, a_t)
$$

取对数后，连乘变连加：

$$
\log P_\theta(\tau)
= \log \rho_0(s_0)
+ \sum_{t=0}^{T-1}
\left[
\log \pi_\theta(a_t \mid s_t)
+ \log P(s_{t+1} \mid s_t, a_t)
\right]
$$

对 $\theta$ 求梯度：

$$
\nabla_\theta \log P_\theta(\tau)
= \sum_{t=0}^{T-1} \nabla_\theta \log \pi_\theta(a_t \mid s_t)
$$

因为：

- 初始状态分布 $\rho_0(s_0)$ 不依赖 $\theta$
- 环境转移概率 $P(s_{t+1} \mid s_t, a_t)$ 也不依赖 $\theta$

环境规则不会因为策略网络参数变了就改变，所以这些项对 $\theta$ 的梯度都是 0。

这说明了一个非常重要的事实：

> 计算策略梯度时，不需要知道环境的转移概率。

这也是为什么策略梯度方法天然是 **model-free** 的：即使环境是黑箱，只要能和它交互、采样轨迹，就能估计梯度。

## 第四步：合并，得到策略梯度定理

把第三步的结果代回第二步：

$$
\nabla_\theta J(\theta)
= \mathbb{E}_{\tau \sim \pi_\theta}
\left[
\sum_{t=0}^{T-1}
\nabla_\theta \log \pi_\theta(a_t \mid s_t)\cdot G(\tau)
\right]
$$

这就是策略梯度定理的基本形式。

实际使用中，通常会把整条轨迹的总回报 $G(\tau)$ 换成从时刻 $t$ 开始往后的回报 $G_t$：

$$
\nabla_\theta J(\theta)
= \mathbb{E}_{\pi_\theta}
\left[
\sum_{t=0}^{T-1}
\nabla_\theta \log \pi_\theta(a_t \mid s_t)\cdot G_t
\right]
$$

原因是：时刻 $t$ 的动作只能影响未来的奖励，不可能影响过去已经发生的奖励。  
所以用 $G_t$ 替换 $G(\tau)$ 不会引入偏差，反而能减少无关噪声。

这就是 **REINFORCE** 算法的核心更新公式。

## 逐项拆解：这个公式在直觉上做什么

看下面这一项：

$$
\nabla_\theta \log \pi_\theta(a_t \mid s_t)\cdot G_t
$$

它可以拆成两个部分理解：

- $\nabla_\theta \log \pi_\theta(a_t \mid s_t)$：告诉我们，应该把参数往哪个方向调，才能让当前选中的动作 $a_t$ 更容易再次被选中
- $G_t$：告诉我们，这次动作带来的结果究竟好不好，并充当这个更新方向的权重

直觉上可以概括成一句话：

> 如果这次动作带来了高回报，就提高它以后被选中的概率；如果这次动作带来了低回报，就压低它以后被选中的概率。

所以策略梯度本质上是在做一种“按回报加权的试错学习”。


## 为什么这个公式方差很大

注意这里的 $G_t$ 是**完整采样出来的蒙特卡洛回报**，通常要等整条轨迹结束后才能计算。  
这意味着 REINFORCE 继承了 MC 方法的典型问题：

- 方差大
- 学习不稳定
- 必须等一整个 episode 结束才能更新

这也就自然引出了后续的改进方法，比如 **Actor-Critic**。

在 Actor-Critic 中，Critic 会学习一个价值函数 $V(s)$ 作为 baseline，从而把更新中的高方差项：

$$
G_t
$$

替换成 Advantage：

$$
A(s_t, a_t) = G_t - V(s_t)
$$

或者进一步用 TD error 来近似。

它们的核心目的都是一样的：在尽量不改变梯度期望的前提下，降低方差，让训练更稳定。

## 一句话总结

策略梯度定理告诉我们：

> 不需要知道环境的转移概率，只要不断采样轨迹，就可以通过  
> $\nabla_\theta \log \pi_\theta(a_t \mid s_t) \cdot G_t$  
> 这样的形式，直接优化策略参数，让高回报动作更容易再次出现。



