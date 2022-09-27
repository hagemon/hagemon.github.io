+++
title="Fastbot2"
date=2022-09-27T22:46:03+08:00
math=true
+++

文章[《Fastbot2: Reusable Automated Model-based GUI Testing for Android Enhanced by Reinforcement Learning》](https://tingsu.github.io/files/ASE22-industry-Fastbot.pdf)的笔记，作者是字节跳动的Zhengwei Lv。

# Introduction

本文介绍了一种基于强化学习模型的自动GUI测试方法，主要思想是利用已经积累的event-activity转移知识（knowledge of historical event-activity transitions），为之后的事件选择做出合理的决策。该方法所要解决的主要问题是：**如何制定事件选择的策略，使得所选择到的事件可以探索到更多的activity，从而能够找到更多潜在的crash。**

本文以覆盖更多activity为目标，提出以下两个方法：

1. 构建全局概率模型，在测试过程中不断更新各事件选择的概率，从而选择能够探索更多页面的事件。
2. 基于N-step SARSA方法，在测试过程中维护Q-table，利用多步信息选择到更合适的事件。

# Workflow

![https://tva1.sinaimg.cn/large/e6c9d24ely1h6l5ikeem5j224q0i4gqo.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h6l5ikeem5j224q0i4gqo.jpg)

本方法主要分为测试前和测试中两个阶段。在测试前，需要安装软件、收集一些静态信息和初始化模型，具体来说：

1. 对安装包进行反编译，获取静态的组件文本标签；
2. 在所有测试机上安装软件；
3. 利用历史数据初始化概率模型（若无则令模型为空，即所有事件都没有被探索过）。

在测试时，需要利用当前的GUI信息进行事件选择、执行、模型更新、Activity覆盖统计和bug统计等，具体来说：

1. 获取当前GUI信息；
2. 获取可被执行的事件；
3. 根据策略选择事件，这些事件应该有更大的概率探索到新的activity；
4. 执行事件；
5. 更新概率模型和Q-table（用于做出更好的决策）；

以上五个步骤在一定时间内会重复执行，直到达到运行时限（如测试1小时）。

# A Throughout Example

本文使用了头条中4个Activity的转换过程为例，介绍了event选择的策略：

<img src="https://tva1.sinaimg.cn/large/e6c9d24ely1h6l94w7andj20u011zahj.jpg" width="50%" style="margin-left:25%">


# Model

## Hyper-events

本文将软件中相似的事件抽象为hyper-event，从而平衡模型的可扩展性和准确性。具体来说，在衡量组件的相似情况时只考虑以下4个因素：

1. 组件所在的Activity；
2. 组件的resource-id；
3. 组件的文本（若是动态获取的文本，则认为它是empty）；
4. 组件支持的事件（如点击和长按）。

若两个组件在这4个因素上相同，则本文认为它们有着相近的功能，并且可以使用同一个hyper-event驱动。

### example

举例来说，例图中的Activity 1中，Title 1和Title 2拥有相同的属性，即：

1. 在同一个activity中；
2. 都具有空文本（动态获取）；
3. 同一个resource-id和同样；
4. 同样的事件类型（点击）。

## Probabilistic Model

概率模型表示了在每个hyper-event转化到各Activity的概率。具体来说有如下定义：

- $\mathcal{E}$表示UI组件中获取到的hyper-events集合；
- $\mathcal{A}$表示待测软件所有的Activity的集合；
- $\delta$表示转移函数，即各hyper-event转移到各Activity的概率，本文用$e\rightarrow (A,p)$表示事件$e$转移到activity $A$ 的概率为$p$。

本文用$M=(\mathcal{E},\mathcal{A},\delta)$表示模型。其中，事件$e$转化为$A_i$的概率计算方法为：

$$
P(e,A_i)=\frac{N(e,A_i)}{N(e)}
$$

即事件$e$转化值$A_i$的次数占事件$e$执行总次数的比例。

模型的构建为事件的选择起到了指导作用，目的是让选择到的事件能够尽可能地探索更多的Activity和潜在的crash。为此，本文进一步提出了基于概率模型（Model-based）和强化学习（Learning-based）的方法。

## Model-based Event Selection

基于概率模型的事件选择方法包含两个模式：模型扩张（expansion）和模型利用（exploitation）。

### Expansion

模型扩张是为了能够探索到更多没有被执行过的事件。如果当前GUI页面的存在从来没有在$M$中出现过的hyper-event，则FASTBOT2会随机从中选一个做为下一步的事件。在模型的初始情况下，或是软件有了新的特征时，会按照这个策略尽快地找到每个页面可执行的事件，并在模型中更新它的转化概率。

### Exploitation

当所有的事件都被执行过一次后，就会进入模型利用模式。FASTBOT2根据已经构建好的概率模型，为当前的页面选择可能覆盖更多activity的事件。具体来说，令$\mathcal{A}_t$为已经被探索过的activity，$\mathcal{E}_c$为当前页面支持的所有事件，则对于事件$e_i$能够探索到的activity个数的期望可以表示为：

$$
\mathbb{E}(e_i)=\sum_{A\notin \mathcal{A_t}}P(e_i,A)
$$

期望越高的事件，应该越容易被选到。于是本文定义事件$e_i$被选择到的概率为：

$$
P_M(e_i)=\exp({\frac{\mathbb{E(e_i)}}{\alpha}})/\sum_{e_i \in\mathcal{E}_c}\exp({\frac{\mathbb{E(e_i)}}{\alpha}})
$$

其中$\alpha$为超参数，用于调节随机性。同时为了公平性，每个事件只能被选取$K$次。本文将两个参数分别设置为0.8和1。

### Example

扩张阶段（Expansion）：假设软件启动于Activity 1，初始状态下所有event都没有被执行过，于是FASTBOT2随机选取一个事件$e$执行，并将其跳转至$A$的情况记录在模型中（即$P(e,A)=1$）。随后在$A$中随机选取未被执行过的事件，持续更新模型。

利用阶段（Exploitation）：对于某个Activity，若尚有未被执行的事件，则依旧按扩张的策略选取；若已经没有未执行的事件，则根据概率选取一个事件，并不断更新概率模型。

举例来说，假设当前的模型状态如例图(b)所示，根据公式可以得到在Activity 1中$e_1$被选中的概率最大，若此时它跳转至Activity 2（这也是概率更高的结果），会发现$e_5$尚未被执行，FASTBOT2会根据扩张策略选择$e_5$。在这个过程中，模型也会更新每个事件被选择的概率。

然而，概率模型方法的缺点是只能通过一步的信息来做出决策，这样容易选择到当前收益高的事件（如执行某个事件，可能探索到$a$个Activities），而无法探测那些后期有更高收益的事件（如执行事件$e_1$并不会带来很大的收益，但在按顺序执行事件$e_1$、$e_3$、$e_8$后会探索到远大于$a$的Activities）。

## Learning-based Event Selection

为了解决上述的问题，本文提出了基于强化学习的方法。具体来说，本文使用了N-Step SARSA的思想，利用Q-table保存每一个事件所能带来的期望收益，这个收益包含了未来$n$步的潜在信息。

基于学习的事件选择方法包也含了模型扩张（expansion）和模型利用（exploitation）两个模式。

### Expansion

在模型扩张时，事件的选择并不会依赖于Q-table，这里可能使用和Model-based一样的扩张方式（选择未被选择过的事件），也可以是完全随机选择。无论在什么样的事件选择策略下，都需要更新Q-table的值。Q-table可以被初始化为一个全0表。

具体来说，当根据某种策略得到了一个“事件-Activity”的序列后，以$n$为窗口大小，更新Q-table的值：

$$
Q(e_t)=Q(e_t)+\alpha(G_{t,t+n}-Q(e_t))
$$

这其实是一种线性插值的形式，其中$G_{t,t+n}$表示未来$n$步的收益：

$$
G_{t,t+n}=r_{t+1}+\gamma r_{t+2}+\dots + \gamma^n Q(e_{t+n})
$$

可以看到，这里使用了未来$n-1$步直接收益的指数加权平均和第$n$步的期望收益，其中，$t$时刻的直接收益定义为：

$$
r_{t+1}=\frac{\mathbb{E}(e_t)}{\sqrt{N(e_t)+1}}+\frac{n_h+0.5*n_c+\sum_{e_i\in \mathcal{E}_c }\mathbb{E}(e_i)}{\sqrt{N(A_t)+1}}
$$

右项中的$n_h$表示事件$e_t$执行后页面$A_t$中没有在之前测试中执行的事件数，$n_c$表示当前测试在$A'$中未执行过的事件个数，$N(A_t)$表示$A_t$在当前测试中被访问的次数。

可以看到，当事件$e_t$的期望高且执行次数少时，左项会较大；当$e_t$执行后的页面被探索次数较少、未探索的事件较多，且这些事件也有很高的期望值时，右值较大。这里分母的根号是为了平衡和分子的量纲差距。

### Exploitation

当所有的事件都被探索之后，可以利用Q-table来进行事件选择的决策。与Model-based类似，每个事件被选中的概率为：

$$
P_Q(e_i)=\exp({\frac{{Q(e_i)}}{\beta}})/\sum_{e_i \in \mathcal{A}_c}\exp({\frac{{Q(e_i)}}{\beta}})
$$

### Example

假设在Activity 2中，$e_4$和$e_6$在本次测试中暂未被执行过，且$e_6$已经加入$M$。在此情况下，在Activity中执行$e_1$有比较高的奖励（因为$n_h$和$n_c$比较大）。类似的，$e_8$作为不在$M$中的新事件，也导致了$e_5$有较高的奖励。在这些过程中，Q-table都在被不断地更新，但没有被使用。当所有事件都被执行过之后，Q-table来时为事件选择做决策，这样可以找到更具价值的事件。

# Evaluation

作者在头条和抖音的10个版本中，与STOAT、APE和MONKEY进行比较。本文提出了两个研究问题：

- RQ1：FASTBOT2能否比其它工具拥有更高的功能覆盖率？
- RQ2：model-based和learning-based方法是否能起到作用？（Ablation Study）

对于RQ1，本文通过各工具在10个版本的软件中所覆盖的Activity数量，以及累计的覆盖数量进行横向对比，得到的结果如下图：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h6liyjrdy4j227c0d4q8n.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h6liyjrdy4j227c0d4q8n.jpg)

可以看到FASTBOT2相交其它工具有着较大的提升。其中抖音由于页面更加复杂（包含在线购物和视频流等功能），FASTBOT2的优势更加明显。

下图还以维恩图的形式展示了FASTBOT2与其它工具的重合程度。

![https://tva1.sinaimg.cn/large/e6c9d24ely1h6lj0imvotj212808gjsd.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h6lj0imvotj212808gjsd.jpg)

对于RQ2，本文分别比较了单独使用概率模型（PM）、单独使用强化学习模型（RL）和联合两个模型（RL+PM）的Activity覆盖率情况，结果如图所示：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h6lj2jr58qj212e0g6tag.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h6lj2jr58qj212e0g6tag.jpg)

可以看到两者的联合有着很大的提升。由于在初始阶段，Q-table的值还未能充分体现真实期望，RL的效果相对较差，当后期Q-table的值较为理想时，效果会超过PM。若在初始阶段使用PM进行事件选择，同时对Q-table进行学习，则在后期使用Q-table做为事件选择依据，可以有比较好的提升。

# Thoughts

我认为本文有以下几个亮点：

1. 相对于传统的Fuzzing策略，本文加入了概率模型和强化学习模型，为事件选择提供了更丰富的先验知识，“探索更多页面”的目的也使得事件选择更具有方向性。
2. Expansion的策略让事件的探索可以提前铺张开，在早期阶段就可以先预估所有事件带来的影响；而Exploitation的策略很好地利用了这个优势，进一步找到更适合的事件。
3. 本文的实验场景构建的很好，使用了两个现实生活中流量很高的软件进行多版本测试，得到的结果认可度较高。

有什么可以提升的吗：

1. 本文的策略和评价标准都在于探索更多的页面，如果团队有资源的话，是否可以加入用户的使用习惯来提供一些指导，用户使用的逻辑往往也可以为事件的选择提供很好的先验知识，无论是用在模型和Q-table的初始化早期学习的过程中，或许都能提供帮助。