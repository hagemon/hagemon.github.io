+++
title= "Neuron Coverage for Adequacy Testing"
date= 2022-09-03T16:09:59+08:00
math= true
tags = ['Software Analyze and Testing', 'Machine Learning Testing']
+++

神经覆盖率的概念首先在DeepXplore的论文《**DeepXplore: Automated Whitebox Testing of Deep Learning Systems**》中被提及，其核心思想是输入数据中被激活的神经元个数占总神经元个数的比例。Lei Ma在此基础上做出了一些改进，提出DeepGauge和Combinatorial；Xiaoning Du提出了针对基于状态的神经网络（如RNN）的覆盖率准则以及相应的工具DeepCruiser；Zhiyang Zhou考虑了特征之间的连接权重，提出了Contribution Coverage和工具DeepCon。

# DeepXplore

深度学习系统被广泛应用在一些安全相关的场景中（如自动驾驶），系统在一些边界情况下（corner cases）的准确性和可预测性是十分重要的。大多深度学习系统依赖于标注的数据，从而在一些边界情况下容易暴露问题，从而引发严重的后果。

于是有以下两个问题需要考虑：

1. 是否有一个标准，可以衡量输入数据的全面性，从而推断模型是否可以考虑到所发生的各种情况。
2. 如何在不需要大量的带标签数据的情况下，可以找到深度学习系统的错误行为。

为了解决这些问题，作者做出了以下几点尝试：

1. 提出神经元覆盖率的概念，用于衡量深度学习系统的逻辑覆盖程度。
2. 基于交叉引用的思想，利用多个相同功能的深度学习模型，在没有标注数据的情况下以投票的形式找到可能存在错误行为的模型。
3. 基于深度学习系统的输出的可微性，构建生成测试数据的联合优化方法，通过链式法则对输入数据$x$进行梯度提升更新，可以在**最大化神经元覆盖率的同时让模型表现出尽可能多的行为**。

## 神经元覆盖率（Neuron coverage）

测试数据集合在神经网络中激活的神经元个数占总神经元个数的比例，被称为该测试数据集合的神经元覆盖率。其中，一旦某个神经元对于某个输入的激活值大于某个值，则认为该神经元被激活。

假设神经网络的神经元集合为$N=\{n_1,n_2,\dots\}$，测试集集合为$T=\{x_1,x_2,\dots\}$，$out(n,x)$表示神经元$n$在对数据$x$进行运算后的输出结果，$t$为神经元被激活的阈值，则神经元覆盖率可以被表示为：

$$
NCov(T,x)=\frac{|n|\forall x\in T,out(n,x)>t|}{N}
$$

## 梯度

与深度学习系统中优化模型的方式不同，作者固定神经元的参数$\theta$，通过计算神经元输出$y$相对于输入$x$的梯度：

$$
G=\nabla_x f(\theta,x)=\partial y/\partial x
$$

## 优化目标

### 最大化不同行为

最大化不同行为指的是生成的数据集可以让每一个模型都尽可能地预测出多种结果。假设有$n$个深度神经网络$F_{k\in 1\dots n}:x\rightarrow y$，若输入$x$在所有的深度神经网络中都可以预测出正确的结果，我们的目的就是在修改$x$后让至少一个深度神经网络有不同的分类结果。

令$F_k(x)[c]$表示第$k$个神经网络将$x$分类为$c$的概率，选择任一神经网络$F_j$，则最大化不同行为的优化目标表示为：

$$
obj_1(x)=\sum_{k\ne j}F_k(x)[c]-\lambda_1\cdot F_j(x)[c]
$$

即对于类别$c$，在最小化$F_j$对其分类的概率的同时，最大化其余所有神经网络对其的分类概率。

### 最大化神经元覆盖率

对于那些激活值小于$t$的神经元，希望它们的值可以大于$t$，于是在优化目标中加上：

$$
obj_2(x)=\lambda_2\cdot f_n(x)
$$

### 联合优化目标

将两个优化目标合并，得到联合优化目标：

$$
obj_{joint}(x)=\sum_{k\ne j}F_k(x)[c]-\lambda_1\cdot F_j(x)[c]+\lambda_2\cdot f_n(x)
$$

其中，$n$是每个iteration中随机选择的非激活神经元。

### 特定域约束

对于特定的问题，数值的范围需要有特定的约束。如图像数据的范围只能在0到255之间。具体来说，输入的更新可以表示为$x_{i+1}=x_i+s\cdot g$，其中$g$为梯度值。作者需要保证在这个更新之后的数据仍然满足约束条件：

- 对于离散特征来说，在更新时需要将$g$四舍五入为整数
- 对于图像特征，有以下三种约束：
  1. 用于模拟不同亮度光亮的光照效果，为所有像素增加或减少一个特定的值。
  2. 使用一个小方块模拟遮挡效果。
  3. 使用多个黑色小方块模拟脏镜头效果。
- 对于安卓应用数据集Drebin，只增加manifest file的特征，而不改变代码本身。

### 算法流程

算法流程可以用以下python伪代码表示：

```python
gen_test = [] # 初始化生成的测试集为空
for x in seed_set.get():  # 无限获取x
		# 对于seed，所有分类器dnns应该都正确分类为类别c
		c = dnns[0].predict(x)
		d = random_pick(dnns)  # 随机选取dnn
		while True:
				obj1 = obj_func_1(x, d, c, l1)  # l1为lambda1
				obj2 = obj_func_2(x, dnns, inactive_neurons)
        obj = obj1 + l2*obj2
        grad = get_grad(obj, x)  # 求obj相对于x的偏导数
        grad = apply_constraint(grad)  # 执行特定域约束规则
        x = x + s * grad
        if d.predict(x) != (dnns-d).predict(x):
            # 随机选取的dnn与其它dnn预测结果不同，可以作为新的数据
            gen_test.append(x)
            # 更新神经元激活状态
            inactive_neurons = update_neurons(inactive_neurons)
						break
		if SATISFIED(inactive_neurons):
        # 如果当前状态满足条件
        return gen_test

def obj_func_1(x, d, c, l1):
    rest = dnns-d
    loss = 0
    for d in rest:
        loss += prob(dnn, c, x)  # 获取x被分类为c的概率
    loss -= l1 * prob(d, c, x)  # 获取x被d分类为c的概率
    return loss

def obj_func_2(x, dnns, inactive_neurons):
    loss = 0
    for d in dnns:
        n = random_pick(inactive_neurons)
        loss += n.foward(x)  # 获取神经元n对x的输出
    return loss
```

# DeepGauge

DeepGauge提出了多个层级的测试覆盖率标准，来检测系统的错误行为。作者认为深度学习系统与传统软件系统一样包含主要功能行为（major function behaviors）和边界行为（corner-case behaviors)，这两类行为都可能包含错误行为。

定义$N=\{n_1,n_2,\dots\}$为神经网络的神经元集合，$T=\{x_1,x_2,\dots\}$为测试集数据，$\phi(x,n)$为神经元$n$对于输入数据$x$的输出，$L_i$为第$i$层的所有神经元构成的集合。

## 神经元级别覆盖率准则（Neuron-Level Coverage Criteria）

在神经元级别的覆盖率中，使用经过训练集训练的神经元权重来计算测试集输出，因此，神经元的输出应该在训练集的影响下符合某种数据分布。我们可以从这个数据分布中知道每个神经元的主要功能和边界功能。

但精确衡量每个神经元的数据分布是十分消耗资源的，因此本文基于数据集来估计每个神经元对于主要功能区域和边界功能区域的边界（boundary）。具体来说，对于训练集中的数据，每个神经元的输出上下界$\rm{high}_n$和$\rm{low}_n$之间的范围$[\rm{low}_n,\rm{high}_n]$为神经元的主功能区域（major function region）。

### $k$单元神经覆盖率（$k$-multisection Neuron Coverage)

为了全面地评估主功能区域，作者将该区域平均划分为$k$个单元，并计算数据在每个单元的覆盖情况，称为$k$单元神经覆盖率。

对于神经元$n$，定义$S_i^n$为落在神经元$n$的第$i$个单元的数据集合，若$\phi(x,n)\in S_i^n$，则称$x$被第$i$个单元覆盖。对于测试集$T$和神经元$n$来说，$k$单元神经覆盖率被定义为被覆盖的单元数占总单元数的比例，即：

$$
\frac{|\{S_i^n | \exists x \in T:\phi(x,n)\in S_i^n\}|}{k}
$$

而对于整个神经网络来说，$k$单元神经覆盖率为所有神经元的覆盖单元数占所有神经元单元总数的比例，即：

$$
\rm{KMNCov}(T,k)=\frac{\sum_{n\in N}|\{S_i^n | \exists x \in T:\phi(x,n)\in S_i^n\}|}{k\times |N|}
$$

### 神经元边界覆盖率（Neuron Boundary Coverage）

对于主功能区域外的区域，即$(-\infty, \rm{low}_n)\bigcup(\rm{high}_n,+\infty)$，称为边界区域（corner-case region）。

对于神经元$n$，若$\phi(x,n)$在边界区域，则认为它被边界区域覆盖，分别定义上边界覆盖（UpperCornerNeuron，UCN）和下边界覆盖（LowerCornerNeuron，LCN）为：

$$
UCN=\{n\in N|\exists x \in T:\phi(x,n)\in (\rm{high}_n,+\infty)\}
$$

$$
LCN=\{n\in N|\exists x \in T:\phi(x,n)\in (-\infty, \rm{low}_n)\}
$$

而神经元边界覆盖率（NBC）则为两者的数量占总覆盖情况数（每个神经元有两种情况，即$2|N|$）的总和：

$$
\rm{NBCov}(T)=\frac{|UCN|+|LCN|}{2\times|N|}
$$

### 强神经元激活覆盖率（Strong Neuron Activation Coverage）

在一些可解释性研究中，神经元的高激活值可能包含它所学到的更多信息，所以定义强神经元激活覆盖率（SNAC）表示上边界覆盖的情况，即

$$
\rm{SNACov}(T)=\frac{|UCN|}{|N|}
$$

## 层级别覆盖率准则（Layer-Level Coverage Criteria）

在层级别，本文使用了最活跃的神经元以及它们的组合来表示深度神经网络的行为。

假设测试数据为$x$，$n_1$和$n_2$分别表示某一层的两个神经元，若$\phi(x,n_1)>\phi(x,n_2)$，则称$n_1$比$n_2$在x上活跃。对于第$i$层，用$\rm{top}_k(x,i)$表示对于$x$来说最活跃的$k$个神经元。

### Top-$k$ Neuron Coverage

Top-k神经元覆盖率（TKNCov）表示对于每一层中的神经元，被至少选中进一次top-k激活值的神经元个数占总神经元个数的比例，即：

$$
\rm{TKNCov}(T,k)=\frac{|\bigcup_{x\in T}(\bigcup_{i\le i \le l}\rm{top}_k(x,i))|}{|N|}
$$

### Top-$k$ Neuron Patterns

在神经网络中，同一层的神经元可能在神经网络判别过程中扮演着相似的角色，而一层中最活跃的神经元可以表示为神经网络的一种模式（pattern）。本文定义，深度神经网络在测试集中至少被认定为一次top-k的神经元的个数，为神经网络的top-k神经元模式：

$$
\rm{TKNPat}(T,k)=|\{top_k(x,1),\dots,top_k(x,l)|x\in T\}|
$$

直观来说，top-k神经元模式基于每一层最活跃的神经元来表示神经网络在多个场景的活跃程度。

## 与DeepXplore的区别

作者通过在原始数据和对抗数据集上，对不同的阈值的DeepXplore神经元覆盖率（DNC）进行实验：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h5ou1gjnosj212t0u0n1p.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h5ou1gjnosj212t0u0n1p.jpg)

可以看到DNC在原始数据和对抗数据的表现具有很高的一致性，缺乏对更多的数据评价的能力。作者还提出了DNC的几点限制：

1. 对于所有神经元，设置的阈值都相同，而实际上不同神经元的数据分布是不一样的。
2. 在判断神经元是否被激活前，需要对利用该层的最大、最小值对神经元的输出进行min-max归一化，使其被映射到$[0,1]$范围内。这样对于不同两个输入$x_1$和$x_2$来说，神经元的不同输出$y_1$和$y_2$有可能会在归一化后被映射到同一个值$y$，而$x_1$和$x_2$本应该包含不同的信息。

# Combinatorial Testing for DL

神经网络的状态可以由神经元的输出来表示，而由于神经元的输出空间很大，无法针对所有的情况进行测试。本文将联合测试（Combinatorial Testing，CT）与深度学习测试相结合，提出了几个测试覆盖率标准，和一个基于CT的测试样例生成技术。

## Combinatorial Testing

在传统的软件测试中，各种各样的配置文件和输入域使得系统质量和准确性的评估具有一定困难。通常来说，软件测试需要考虑到所有可能的输入，但这在实际运行的时候是基本不可能的。所以需要一个采样方法来减少输入数据的维度，我们称为基于联合测试（CT）的黑盒采样。

大部分的软件问题是由于几个不同的因素组合而造成的（如当操作系统为MacOS，同时屏幕分辨率为1080*1920时才会出现bug），组合式测试的核心思想就是找到能够用最少的组合来覆盖多个因素的测试用例。

## Combinatorial Testing Criteria for DL

定义$N=\{n_1,n_2,\dots\}$为神经网络的神经元集合，$T=\{x_1,x_2,\dots\}$为测试集数据，$\phi(x,n)$为神经元$n$对于输入数据$x$的输出，$L_i$为第$i$层的所有神经元构成的集合。对于输入$x$激活值大于0的神经元$x_i$，将其标注为活跃神经元，用$A(n_i,x)=1$表示，反之用$A(n_i,x)=0$表示非活跃神经元。

### 神经元激活配置（Neuron-activation configuration）

对于包含$k$个神经元的集合$L_i$和测试数据$x$，一个神经元配置$c=(b_1,b_2,\dots,b_k)$表示每个神经元对于测试数据的激活情况，其中$b_i\in \{0, 1\}$。本文使用$\rm{FC\_NA(T,L_i)}=1$表示$L_i$的所有配置（所有可能的$k$元组）都能被测试集$T$覆盖，反之$\rm{FC\_NA(T,L_i)}=0$；用$\rm{NC\_NA(T,L_i)}$表示$L_i$被测试集$T$所覆盖的神经元激活配置个数。

### $t$路组合稀疏覆盖率（$t$-way combination sparse coverage）

$t$路组合稀疏覆盖率用于减少测试深度学习系统时的测试配置数量。给定一个数据集$T$和神经元集合$L_i$，使用$\Theta$来表示$L_i$中所有的t路神经元组合（即所有t个神经元组成的t元组的集合）。$t$路组合稀疏覆盖率表示$L_i$的所有的t路神经元组合中，能够在$T$上包含全部配置的t路神经元组合比例，即

$$
\rm{TWsCov}(T,L_i)=\frac{|\{\theta\in \Theta|\rm{FC\_NA(T,\theta)}\}|}{|\Theta|}
$$

举例来说，若$L_i=\{n_1,n_2,n_3\}$，则有3个2路神经元：$\Theta=\{\{n_1,n_2\}\cup\{n_1,n_3\}\cup\{n_2,n_3\}\}$。每一个2路神经元包含4种配置，$c_1=(0,0)$、$c_2=(0,1)$、$c_3=(1,0)$、$c_4=(1,1)$。假设$T$拥有5个样本，且在$L_1$的输出如下表所示：

| $n_1$    | $n_2$    | $n_3$    |
| ---- | ---- | ---- |
| 0    | 0    | 1    |
| 0    | 1    | 0    |
| 1    | 0    | 1    |
| 0    | 0    | 1    |
| 1    | 1    | 1    |

则只有$\{n_1,n_2\}$能够包含所有配置，即$\rm{FC\_NA(T,\{n_1,n_2\})}=1$，于是在$T$下$L_1$的2路组合稀疏覆盖率为$1/3$。

### $t$路组合稠密覆盖率（$t$-way combination dense coverage）

由于$\rm{TWsCov}$并不能将所有的组合情况都考虑进去，本文提出了$t$路组合稠密覆盖率$\rm{TWdCov}$。

与$\rm{TWsCov}$只考虑满足所有配置的组合不同，$\rm{TWdCov}$考虑了所有出现的组合占组合总数的百分比，其中$\rm{NC\_NA}(T,\theta)$表示神经元组合$\theta$（如$\{n_1,n_3\}$）在数据集$T$中出现过的配置个数，以上表为例就是$\rm{NC\_NA}(T,\{n_1,n_3\})=|\{(0,1),(0,0),(1,1)\}|=3$，而所有神经元组合可能出现的配置总数为$2^t|\Theta|$。所以$\rm{TWdCov}$被表示为：

$$
\rm{TWdCov}(T,L_i)=\frac{\sum_{\theta\in\Theta}\rm{NC\_NA(T,\theta)}}{2^t|\Theta|}
$$

### $(p,t)$-完整性覆盖率（completeness coverage）

$(p,t)$-完整性覆盖率表示稠密覆盖率高于$p$的神经元组合占组合总数的百分比。举例来说：

- $\{n_1,n_2\}$的所有配置都被激活了，稠密覆盖率为1。
- $\{n_1,n_3\}$和$\{n_2,n_3\}$各自被激活了3个配置，稠密覆盖率为0.75。

则$L_i$对于测试集$T$的$(0.7,2)$-完整性覆盖率为1（即都超过0.7），$(0.9,2)$-完整性覆盖率为$1/3$。

上述几种覆盖率都是针对某一个网络层，而对于神经网络可以在每一层中都计算对应的覆盖率，来观察每一层的测试覆盖效果。

# DeepCruiser

## RNN中的状态转移建模

### RNN的隐含状态和状态转移

DeepCruiser为基于状态的神经网络（如RNN）提出了覆盖率标准。以RNN为例，RNN的输入为一个序列数据$x\in X^N$，其中$N$为序列长度，$x_i$为序列的第$i$个元素。RNN中会维护一个状态向量$s\in S^N$，$s_i$表示第$i$个状态，它可以是一个$m$维的向量。在训练过程中，RNN会基于上一个状态向量与当前的元素相结合，计算出当前时刻的输出与下一个状态向量，即$(s_{i+1},y_i)=f(s_i,x_i)$。初始状态下$s_0=\bf{0}$，即全0向量。对于这样的输入$x$，会产生一个从$s_0$到$s_n$的有限转移序列$t$。

对于两个序列输入$x_0x_1x_2$和$x_0x_1'x_2'$，可以分别产生输出$y_0y_1y_2$和$y_0y_1'y_2'$，中间的状态分别为$s_0s_1s_2s_3$和$s_0s_1s_2's_3$，可以用下图来表示

![https://tva1.sinaimg.cn/large/e6c9d24ely1h5r3pvqfzij20s60bc754.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r3pvqfzij20s60bc754.jpg)

### 抽象状态转移模型

RNN中的状态序列$S$可能是一个很大的序列，因此需要对每一个状态和状态之间的转移进行抽象，从而减少样本的搜索空间。

对于**状态（state）的抽象**来说，本文将每一个$m$维的状态向量通过PCA降至$k$维。再分别获取这$k$个维度的数值上界$up_d$和下界$ld_d$，并以此将每个维度再平均划分为$m$个等宽区间。根据状态向量$s$分别在$k$个维度中的索引，确定它的抽象状态$\hat{s}$。对于两个抽象状态之间的距离，计算它们每个维度的索引之差的绝对值并求和：

$$
d(\hat{s},\hat{s'})=\sum_{d=1}^k|I^d(\hat{s})-I^d(\hat{s'})|
$$

因此，很多距离相近的状态向量会被映射到同一个抽象表示中，这样大大减少了数据的维度。

对于**转移（transition）的抽象**来说，在确定了状态的抽象之后，即可以把原本的状态转移（如$(s_1,s_2)$）代入至抽象后的状态（如$(\hat{s_1},\hat{s_2})$。

### 将训练后的RNN转化为马尔可夫决策过程

作者通过统计从$\hat{s}$到$\hat{s'}$的状态转移频率，作为状态转移概率$Pr_{\hat{x}}(\hat{s},\hat{s'})$：

$$
Pr_{\hat{x}}(\hat{s},\hat{s'})=\frac{\{|(s,s',x)|s\in \hat{s}\wedge x\in \hat{x}\wedge s'\in \hat{s'}\}|}{|(s,\_,x)|s\in\hat{s}\wedge x\in\hat{x}|}
$$

## RNN的覆盖率准则

令$M=(\hat{S},I,\hat{T})$表示RNN抽象的马尔可夫决策过程，$\hat{S}$为抽象的状态集合，$I$为初始状态集合，$\hat{T}$为状态转移概率分布，$T=\{x_0,\dots,x_n\}$为测试集。值得注意的是原始状态和抽象状态都是从训练集中获取的。

### 状态级别覆盖率准则（State-Level Coverage Criteria）

与$k$单元神经元覆盖率类似，状态级别的覆盖率包含主功能区域和边界区域，对于所有维度$k$的每一个元素$d$，落在在$up_d$和$ld_d$之间的称为主功能区域，反之为边界区域。

- 基本状态覆盖率：测试集的抽象状态$\hat{S}_T$与训练集的抽象状态$\hat{S}_M$的交集数量站训练集抽象状态的比例：

  $$
  \rm{BSCov}(T,M)=\frac{|\hat{S}_T\cap\hat{S}_M|}{|\hat{S}_M|}
  $$

- $k$步状态边界覆盖率（$k$-step state boundary coverage）：对于测试集中不在主功能区域的状态，可以计算这些状态与训练集状态之间的距离。$k$步状态边界覆盖率表示测试集状态中，存在与训练集状态距离小于$k$的状态数的比例，即：

  $$
  \rm{k-SBCov}(T,M)=\frac{|\hat{S}\_T \cap \bigcup\_{i=1}^k \hat{S}\_{M^c}(i)|}{|\hat{S}\_{M^c}(i)|}
  $$

  其中$\hat{S}_{M^c}(i)$表示与训练集状态的最小距离等于$i$的状态集合。

### 转移级别覆盖率准则

- 基本转移覆盖率：测试集的状态转移$\hat{\delta}_T$与训练集的状态转移$\hat{\delta}_M$的交集数量站训练集状态转移的比例：

  $$
  \rm{BTCov}(T,M)=\frac{|\hat{\delta}\_T\cap\hat{\delta}\_M|}{|\hat{\delta}\_M|}
  $$

- 输入空间覆盖率：训练集和测试集可能会拥有相同的抽象状态，输入空间覆盖率衡量了测试集与训练集共享的抽象状态中，对应的测试集输入占训练集所有状态输入的比例，即：

  $$
  \rm{ISCov}(T,M)=\frac{\sum\_{\hat{s}\in\hat{S}\_T\cap\hat{S}\_M}|\hat{X}\_T(\hat{s})|}{\sum\_{\hat{s}\in\hat{S}\_M}|\hat{X}\_M(\hat{s})|}
  $$

- 加权输入覆盖率：基于训练集与测试集相交的抽象状态与其对应的同样相交的输入数据，获取相应的状态转移概率并相加，结果值占训练集状态对应的输入数量的比例。对于相交状态来说，其出现概率越高，则覆盖率就越大：

  $$
  \rm{WICov(T,M)}=\frac{\sum\_{\hat{s}\in\hat{S}\_T\cap\hat{S}\_M}\sum\_{\hat{x}\in\hat{X}\_T(\hat{s})\cap\hat{X}\_M(\hat{s})}\sum\_{\hat{s'}\in \hat{\delta}\_T(\hat{s})}\hat{T}(\hat{s},\hat{x},\hat{s'})}{\sum\_{\hat{s}\in \hat{S}_M}|\hat{X}_M(\hat{s})|}
  $$

## 蜕变测试

本文以音频翻译为场景，利用蜕变测试的思想，对输入的音频进行一系列约束范围内的转换，使得变体样本$a'$和原始样本$a$一样可以被人类识别。

将$a'$和$a$同时翻译后，比较二者的差异：

- 若差异超过一定阈值，则认为$a'$是一个failed test，将其加入失败测试用例中
- 若$a'$使得覆盖率提升，则认为$a'$是一个有益的样本，将其加入测试集中。

# MC/DC Coverage Criterion for DL

## MC/DC

MC/DC的核心思想是，对于一个包含多个条件的决定，条件中所有可能的因素都必须被测试到，如：

```python
if (a > 3 or b = 0) and c != 4:  # 3个条件
    pass
```

则以下4个测试用例可以覆盖3个条件的所有情况：

```
1. (a>3)=false, (b=0)=true, (c!=4)=false
2. (a>3)=true, (b=0)=false, (c!=4)=true
3. (a>3)=false, (b=0)=false, (c!=4)=true
4. (a>3)=false, (b=0)=true, (c!=4)=true
```

其中，前两个测试用例满足条件覆盖（即每一个条件的true和false都出现过）和决定覆盖（所有的决定都出现过，即整个if语句的结果中true与false都出现了）。而DC/MC的要求中，每一个条件都应该单独影响一次决定，即在仅改变某个条件的情况下，决策结果发生变化。

## 神经网络中的条件与决策

对于神经网络来说，第$k$层第$l$个神经元的输入$u_{k,l}$为：

$$
u_{k,l}=b_{k,l}+\sum_{1\le h \le s_{k-1}}w_{k-1,h,l}\cdot o_{k-1,h}
$$

其中，$w_{k-1,h,l}$表示第$k-1$层第$h$个神经元和第$k$层第$l$个神经元之间的权重，$o_{k-1,h}$表示第$k-1$层第$h$个神经元的输出。

可以看到，其结构与传统软件的if语句类似，其中$w_{k-1,h,l}\cdot o_{k-1,h}$表示每个条件的值，$u_{k,l}$表示该神经元的决策。

令$\Psi_k$为第$k$层节点子集的集合，它可以包含该层中任意个节点的组合。$\Psi_k$中的每一个元素$\psi_{k,i}$可以看作一个决策，它包含了若干个条件（即节点），这些条件可以与第$k-1$层的决策相连接。

令$(\psi_{k,i},\psi_{k+1,j})$为相邻的两组特征，对于深度神经网络$\mathcal{N}$，其所有的相邻特征组为$\mathcal{O(N)}$。本文定义，若神经元$n$在输入$x$时被ReLU函数激活，则$sign(n,x)=1$，否则$sign(n,x)=-1$。

### 符号变化（sign change）

对于特征$\psi_{k,l}$和两个输入$x_1$与$x_2$，定义**符号变化**（sign change，sc）和非符号变化（not sign change，nsc）分别为：

- 对于所有的$n_{k,j}\in \phi_{k,l}$有$sign(n_{k,j},x_1)\ne sign(n_{k,j},x_2)$，记为$sc(\psi_{k,j},x_1,x_2)$。
- 对于所有的$n_{k,j}\in \phi_{k,l}$有$sign(n_{k,j},x_1)= sign(n_{k,j},x_2)$，记为$nsc(\psi_{k,j},x_1,x_2)$。

### 值变化（value change）

定义一个抽象的映射$g$，使得$g(\psi_{k,l},x_1,x_2)\rightarrow\{true,false\}$。举例来说，这里的$g$可以是一个距离函数，来判断两个输入样本在当前特征上的距离是否大于阈值，从而体现两个样本的输出特征具有一定差异。如：

```python
def g_demo(psi, x1, x2):
    v1 = get_value(psi, x1)  # 获取x1在psi对应神经元输出
    v2 = get_value(psi, x2)
		dist = some_distance(v1, v2)
    return dist 》 0.1
```

对于特征$\psi_{k,l}$、两个输入$x_1$与$x_2$和映射$g$，定义**值变化**（value change，vc）为：

- $g(\psi_{k,l},x_1,x_2)=true$

记为$vc(g,\psi_{k,l},x_1,x_2)$若条件不满足，则定义为$\neg vc(g,\psi_{k,l},x_1,x_2)$。

## Covering Methods

### Sign-Sign Coverage（SS Coverage）

假设特征对$\alpha=(\psi_{k,i},\psi_{k+1,j})$和测试用例$x_1$与$x_2$，若符合以下条件：

- $sc(\psi_{k,i},x_1,x_2)$且$sc(P_k \backslash \psi_{k,i},x_1,x_2)$；
- $sc(\psi_{k+1,j},x_1,x_2)$

则称$\alpha$对于$x_1$与$x_2$满足SS覆盖，记为$SS(\alpha,x_1,x_2)$。其中$P_k$为第$k$层所有节点。

条件1说明了对于$x_1$与$x_2$，特征$\psi_{k,i}$的激活情况完全不同，且除了$\psi_{k,i}$以外的节点激活情况完全相同，即覆盖了$\psi_{k,i}$的不同条件；条件2说明了$x_1$与$x_2$可以使$\psi_{k,i}$特征作为condition独立影响$\psi_{k+1,j}$的decision，这使得SS很接近MC/DC的含义，但是以特征为粒度，而不是以节点为粒度。

### Value-Sign Coverage（VS Coverage）

假设特征对$\alpha=(\psi_{k,i},\psi_{k+1,j})$和测试用例$x_1$与$x_2$，若符合以下条件：

- $vc(g,\psi_{k,l},x_1,x_2)$且$nsc(P_k,x_1,x_2)$；
- $sc(\psi_{k+1,j},x_1,x_2)$

则称$\alpha$对于$x_1$与$x_2$满足VS覆盖，记为$VS(\alpha,x_1,x_2)$。

具体来说，$k$层节点的所有激活状态都没有改变，但特征$\psi_{k,i}$在两个样本下的值有差异，从而导致特征$\psi_{k+1,j}$的激活状态改变。

### Sign-Value Coverage（SV Coverage）

假设特征对$\alpha=(\psi_{k,i},\psi_{k+1,j})$和测试用例$x_1$与$x_2$，若符合以下条件：

- $sc(\psi_{k,i},x_1,x_2)$且$sc(P_k \backslash \psi_{k,i},x_1,x_2)$；
- $vc(g,\psi_{k+1,j},x_1,x_2)$且$nsc(\psi_{k+1},x_1,x_2)$；

则称$\alpha$对于$x_1$与$x_2$满足SV覆盖，记为$SV(\alpha,x_1,x_2)$。

SV Coverage反映了特征$\psi_{k,i}$的激活情况改变时，特征$\psi_{k+1,j}$的值发生变化，即条件改变导致决策的值发生变化的情况。

### Value-Value Coverage（VV Coverage）

假设特征对$\alpha=(\psi_{k,i},\psi_{k+1,j})$、测试用例$x_1$与$x_2$，与两个映射函数$g_1$和$g_2$，若符合以下条件：

- $vc(g_1,\psi_{k,i},x_1,x_2)$且$nsc(P_k,x_1,x_2)$；
- $vc(g_2,\psi_{k+1,j},x_1,x_2)$且$nsc(\psi_{k+1},x_1,x_2)$；

则称$\alpha$对于$x_1$与$x_2$满足VV覆盖，记为$VV(\alpha,x_1,x_2)$。

在特征$\psi_{k,i}$值改变且$k$层神经元激活状态都不改变的情况下，特征$\psi_{k+1,j}$的值发生变化，且$k+1$层的激活状态不发生改变，则认定为VV覆盖。

# DeepCon

现有大多数方法只使用了神经元输出来衡量覆盖率，DeepCon还使用了神经元输入的权重作为参考。

基于神经元输出的方法中，只要输出值够大，则它被认定为激活神经元的可能性就更大，但若它与下一层神经元连接的权重很小，则这个输出在神经网络中并不会发挥很大的作用，DeepCon因此把权重也作为覆盖率的衡量标准。

## Contribution Coverage

### Contribution Extraction

令$x$为输入数据，$l$为神经网络层数，$n_{i,h}$表示第$i$层第$h$个神经元，$c_{j,h}^i$和$w^i_{j,h}$表示第$i$层第$h$个神经元与第$i-1$层第$j$个神经元相连的关系与权重，则其对$n_{i,h}$的输入为$u_{j,h}^i(x)=w_{j,h}^i\cdot o_{i-1,j}(x)$，其中$o_{i-1,j}(x)$为第$i-1$层第$j$个神经元的输出。第$i$层的第$h$个神经元的所有输入为$U_h^i(x)=\{u_{j,h}^i(x)|0\le j\le s_{i-1}\}$，其中$s_{i-1}$为第$i-1$层的神经元个数。

由于每个神经元的数据分布不同，本文将每个神经元的输入值$u_{j,h}^i(x)$Min-Max归一化为$nu_{j,h}^i(x)$：

$$
nu_{j,h}^i(x)=\frac{u_{j,h}^i(x)-\min(U_h^i(x))}{\max(U_h^i(x))-\min(U_h^i(x))}
$$

若$nu_{j,h}^i(x)$大于阈值$t$，且神经元$n_{i,j}$被认为是激活的（即输出也大于一定阈值），则$c_{j,h}^i$被认为被$x$激活，本文用$isAct(u_{j,h}^i,t,x)$表示是否被激活。另外对于输入层来说，只有输出值最高的神经元被认为是激活的，其余的为不激活的。

为了获取所有被激活的连接关系$c$，从而构成集合$\tilde{U}(x)$，DeepCon通过$x$的预测结果，从最后一层进行反向传播，找到所有被激活的连接$c$，具体来说：

1. 对于输入$x$，找到输出层激活值最高的神经元$n_{l-1,j}$，即神经网络的输出结果。
2. 通过$isAct$函数，找到与$n_{l-1,j}$与$l-2$层中所有相连的关系中被激活的关系，构成$\tilde{U}^{l-1}(x)$，合并至$\tilde{U}(x)$。
3. 从$l-2$层遍历至第一层，对于第$i$层有：
   1. 找到$\tilde{U}^{i+1}(x)$对应的输入神经元。
   2. 通过$isAct$找到这些神经元与上一层的相连关系中，被激活的关系，构成$\tilde{U}^{i}(x)$。
   3. 合并至$\tilde{U}(x)$

最终，Contribution Coverage的定义为被激活的关系数占总关系数的比例，即：

$$
ConC(T)=\frac{|u|\exists x\in T,u\in U, isAct(u,t,x)|}{|U|}
$$

同时，作者还提出了每一层的Contribution Coverage，定义为该层被激活的关系占该层的关系比例：

$$
ConC_i(T)=\frac{|u|\exists x\in T,u\in U^i, isAct(u,t,x)|}{|U^i|}
$$

## DeepCon-Gen

本文还基于contribution coverage的引导进行测试用例的生成，通过梯度上升的方法不断更新输入$x$，使得更新后的$x$可以激活之前未激活的相连关系。

具体来说，DeepCon-Gen随机选取$x$在神经网络中未激活的相连关系，最大化该关联输出给神经元$n$的值$u$与神经元$n$的输出值$o$。但在优化过程中，$x$可能只会通过增加其中一个值就可以达到梯度提升的效果，这样并不能保证该关系被激活。于是作者使用了两个惩罚值，使得一旦神经元的输入值$u$或输出值$o$达到阈值，就停止针对该损失项进行优化，直到两者同时达到阈值。

```python
x = seeds[i]  # 获取输入数据
c = random_pick_inactive_con(model, x)  # 随机选取未激活关系
u, o = get_in_and_out(c)  # 获取该关系对应的神经元的输入和输出
pCon, pOut = MAX_VAL, MAX_VAL

step = 0
while step < MAX_STEP:
    obj = min(u, pCon) + min(o, pOut)
    x = gradAsc(obj, x, eta) # 损失函数关于x的梯度提升更新，步长为eta
    if isAct(u) and pCon == MAX_VAL:
    pCon = min(u, pCon)  # 损失值不会为了增加u而变化，除非u变得小于阈值
    if isAct(o) and pOut == MAX_VAL:
        pOut = min(o, pOut)  # 同理
    if isAct(u) and isAct(o):
        update_cov()  # 更新覆盖率
        tests += x  # 将x添加到生成的测试用例集合中  

```

# Comparison

要比较各个覆盖率的有效性，可以从以下几个方面考虑：

1. 覆盖强度：覆盖率是否能够因为测试集的不断增加而保持增加，而且不会保持在一个非常高的水平
2. 对抗样本的检测：能否通过覆盖率体现出测试集中存在的对抗样本
3. 适用性：是否能够在多种类型的神经网络中保持效果
4. 效率：它计算所消耗的时间是否可以接受

## 覆盖强度

对于覆盖强度来说，若随着测试集的扩大，覆盖率保持不变，则认为它并不能很好地反应测试集的多样性；若覆盖率一直很高甚至到100%，则认为它对测试样例覆盖情况的表达能力有限。

对于模型之间的覆盖强度对比，在同样的测试集下，更好的覆盖率准则应该有这更平滑的上升曲线和相对更低的覆盖率，这样它对未知数据存在更大的空间。

不仅是针对模型之间的覆盖强度，模型内部各个模块的覆盖强度也可以反映出模型的一些特点，如某些网络层可能考虑的因素更多，其覆盖率始终较低，或是其在设计时决定了它并没有利用前一层的很多信息，可以让开发者考虑对其进行优化。

## 对抗样本检测

根据观察，同类别的正常样本倾向于拥有相同的被激活神经元，而对抗样本则倾向于激活不相同的神经元。

令训练集中每个类别的样本所激活的神经元集合为类别神经元（category-neurons）$\tilde{C}$，若测试集中的同类别样本的激活神经元$\tilde{C}(x)$于训练集的类别神经元的交集比例小于一定阈值$\gamma$，则认为$x$是一个对抗样本：

$$
\frac{\{\tilde{C}(x)\}\cap\{\tilde{C_c}\}}{\tilde{C}(x)}
$$

上述规则同样可以扩展到每一层的神经元激活情况，也可以是DeepCon中的激活相连关系。

基于现有的对抗样本生成技术，通过比较不同覆盖率方法对对抗样本检测的ROC曲线，找到效果更好的覆盖率准则。

## 适用性

观察覆盖率准则在不同规模的神经网络中，随着测试集规模增大的变化曲线是否能够拥有正常的增长趋势，并对未知数据留有一定的探索空间。

## 效率

分析覆盖率的计算方式，评估在网络规模的扩大时，计算量是线性增长、指数级增长还是对数级增长。