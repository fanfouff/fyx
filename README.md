# Simulink 模型搭建说明（中文）

## 1. 目标
构建一个用于论文仿真的 Simulink 模型，研究 DoS 攻击下 4 节点 Lorenz 复杂网络的弹性同步：
- 网络动力学对应论文式(1)
- DoS 切换信号 \sigma(t) 对应论文式(2)
- 控制策略包含：无控制 / 线性控制 / 固定时间控制
- 输出指标为总同步误差范数 ||e(t)||

## 2. 顶层结构
建议模型命名为 `resilient_cdn_dos.slx`，顶层包含以下子系统：
1. `Leader (Lorenz)`
2. `Followers Network (4x Lorenz + Coupling)`
3. `DoS Signal Generator`
4. `Controller Bank`
5. `Error Norm Calculator`
6. `Scopes + To Workspace`

信号流向：
- `Leader` 输出主系统状态 s(t)
- `Followers Network` 输出 4 个从节点状态 x_i(t)
- `DoS Signal Generator` 输出 sigma(t)
- `Controller Bank` 接收 e_i(t)=x_i-s 和 sigma(t)，输出控制输入 u_i(t)
- `Followers Network` 接收 u_i(t) 并更新状态
- `Error Norm Calculator` 输出 ||e(t)||

## 3. DoS 信号模块（MATLAB Function）
建议使用 `Clock` + `MATLAB Function`，函数代码如下：

```matlab
function sigma = dos_switch(t, duty, freq, phase)
% sigma = 1: 攻击活跃；sigma = 0: 攻击休眠
T = 1 / freq;
tau = duty * T;
local_t = mod(t + phase, T);
if local_t < tau
    sigma = 1;
else
    sigma = 0;
end
end
```

建议参数：
- duty: 0.20 / 0.35 / 0.50
- freq: 0.50 Hz（可扩展扫描）
- phase: 0

## 4. 控制器模块
输入：
- `e`（12x1，按 [e1; e2; e3; e4] 堆叠，每个 e_i 属于 R^3）
- `sigma`（标量）
- `mode`（0=无控制，1=线性，2=固定时间）

控制律：
- 无控制：u = 0
- 线性：u = (1-sigma) * (-k_lin * e)
- 固定时间：u = (1-sigma) * (-k1*e - k2*sig(e)^p - k3*sig(e)^q)

向量 `sig` 函数：

```matlab
function y = sig_pow(v, r)
y = abs(v).^r .* sign(v);
end
```

## 5. 从节点网络模块
在 `Followers Network` 内：
- 建 4 个 Lorenz 节点（每个节点 3 维状态积分）
- 耦合项采用拉普拉斯矩阵 L：
  xdot_i = f(x_i) - c * sum_j L(i,j) * Gamma * x_j + u_i

推荐环形拓扑拉普拉斯矩阵：

```matlab
L = [ 2 -1  0 -1;
     -1  2 -1  0;
      0 -1  2 -1;
     -1  0 -1  2];
```

Lorenz 动力学：

```matlab
f1 = sigmaL * (x2 - x1);
f2 = x1 * (rhoL - x3) - x2;
f3 = x1 * x2 - betaL * x3;
```

## 6. 误差范数模块
可用 `Sum + Math Function + Sqrt`，也可用 MATLAB Function：

```matlab
function en = error_norm(e)
% e 为 12x1 误差向量
en = sqrt(sum(e.^2));
end
```

## 7. 对比实验设置
每个攻击场景运行三组控制：
1. mode=0（无控制）
2. mode=1（线性控制）
3. mode=2（固定时间控制）

攻击占空比：
- 轻度：20%
- 中度：35%
- 重度：50%

默认攻击频率：0.50 Hz（可做频率扫描）。

## 8. 数据记录与图表
建议使用 `To Workspace` 记录：
- `tout`
- `err_norm`
- `sigma`

后处理建议：
- 用 `semilogy(tout, err_norm)` 画轻/重攻击误差图
- 收敛时间定义为：误差首次小于阈值 0.01 且之后始终不再越界

## 9. 推荐参数
- Lorenz: sigmaL=10, rhoL=28, betaL=8/3
- Network: N=4, n=3, c=1, Gamma=eye(3)
- 线性控制: k_lin=2
- 固定时间控制: k1=2, k2=1, k3=1, p=1.5, q=0.8

## 10. 与 MATLAB 脚本的关系
- `run_resilient_sync_experiments.m`：单次论文对比实验（轻/重攻击 + 收敛时间表）
- `run_batch_experiments_auto.m`：批量自动化实验（占空比/频率扫描 + 自动输出结果）

## 11. 可直接执行的实验命令
在 MATLAB 命令行执行：

```matlab
cd('c:/Users/86156/MATLAB/Projects/fyx_paper/simulation')

% 单次实验：生成轻/重攻击图和基础收敛时间表
run('run_resilient_sync_experiments.m')

% 批量自动化实验：按 dutyList、freqList 自动扫描并保存结果
run('run_batch_experiments_auto.m')
```
