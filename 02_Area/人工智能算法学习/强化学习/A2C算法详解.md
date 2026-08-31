在 [[02_Area/人工智能算法学习/强化学习/强化学习基本概念]] 中提到**Policy-based 方法**直接优化策略 π，用的是策略梯度；**Value-based 方法**学 Q(s,a)，靠"选 Q 最大的动作"来决策。

而 Actor-Critic 方法则是两者的结合：
**Actor（演员）**：一个策略网络 πθ(a∣s)\pi_\theta(a|s)πθ​(a∣s)，负责"做决策"——在状态 s 下应该选什么动作。

**Critic（评论家）**：一个价值网络（学 V 或 Q），负责"打分"——评价 Actor 刚才选的这个动作到底好不好。

## Policy-based的方法的痛点
策略梯度（比如 REINFORCE）的更新公式：
$$
∇_θ​J(θ)=E_π​[∇_θ​logπ_θ​(a_t​∣s_t​)⋅G_t​]
$$
这里的 $G_t$​ ​ 是整条轨迹算出来的**真实回报**（蒙特卡洛法）。问题是：这个 $G_t$​ 方差巨大——因为它是一整条轨迹随机性的累积。方差大意味着训练信号噪声大、收敛慢、不稳定。

Actor-Critic 的核心改进：**把  $G_t$​  换成 Critic 估计出来的价值**，用 TD 的思路（自举）代替蒙特卡洛的完整回报：
$$
∇_θ​J(θ)=E_π​[∇_θ​logπ_θ​(a_t​∣s_t​)⋅Q_w​(s_t​,a_t​)]
$$

这里 QwQ_wQw​ 就是 Critic 学出来的动作价值函数（w 是 Critic 网络的参数）。由于 TD 方法方差更小，训练比纯 REINFORCE 稳定得多.

以上是 MC vs TD 章节的直接应用。


## 更常见的方法：用Advantage代替Q
### 优势函数 Advantage Function
$$
A^π(s,a)=Q^π(s,a)−V^π(s)
$$

含义：这个动作比"这个状态下的平均水平"好多少。如果 A>0A>0A>0，说明这个动作比平均动作强，该多选；如果 A<0A<0A<0，说明这个动作比平均差，该少选。

更新公式变成：
$$
∇_θJ(θ)=\mathbb{E}_π[∇_θlog⁡π_θ(a_t∣s_t)⋅A(s_t,a_t)]
$$
好处是进一步降低方差（减掉一个只依赖状态、不依赖动作的基线 V(s)V(s)V(s)，不会引入偏差，但能让梯度信号更"干净"）。这套框架专门叫 **A2C（Advantage Actor-Critic）**。

实践中 Advantage 常用 **TD error** 来近似（这里就直接用到了贝尔曼方程的自举思想）：
$$
A(s_t,a_t) \approx r_{t+1} + \gamma V(s_{t+1}) - V(s_t)
$$



## 框架
| 算法        | On/Off-policy | 动作空间  | 备注                         |
| --------- | ------------- | ----- | -------------------------- |
| A2C / A3C | On-policy     | 离散为主  | 最经典的同步/异步 AC               |
| DDPG      | Off-policy    | 连续    | AC + replay buffer + 确定性策略 |
| TD3       | Off-policy    | 连续    | DDPG 的改进版，缓解过估计            |
| SAC       | Off-policy    | 连续    | AC + 最大熵正则化，探索性更强          |
| PPO       | On-policy     | 离散/连续 | 严格说也是 AC 结构，只是更新方式加了裁剪约束   |
**AC 既能是 on-policy（A2C/PPO），也能是 off-policy（DDPG/TD3/SAC）**——取决于 Critic 学的是"当前策略的价值"还是"允许用历史数据估计的价值"。如果 Critic 用 replay buffer 训练（意味着允许用旧策略产生的数据），那整个算法就是 off-policy 的；如果每次更新完就必须扔掉旧数据重新采样，就是 on-policy 的。