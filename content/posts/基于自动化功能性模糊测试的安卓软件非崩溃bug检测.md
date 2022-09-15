+++
title = "基于自动化功能性模糊测试的安卓软件非崩溃bug检测"
date = 2022-08-08T21:52:15+08:00
math = true
tags = ['Software Analyze and Testing']
+++

文章[Fully Automated Functional Fuzzing of Android Apps for Detecting Non-Crashing Logic Bugs](https://tingsu.github.io/files/oopsla21-Genie.pdf)的笔记，作者是Ting Su老师。

## 概述

目前的测试工具大多用于检测引起崩溃（crash）的bug，而缺乏一个自动化检测非崩溃性bug（non-crashing bugs）的方法。引发这些非崩溃性bug的主要原因通常是软件的实现逻辑有误，从而导致GUI的显示没有达到预期。

目前要解决非崩溃的功能性问题，主要面临两个挑战：

1. 需要大量的经验和人力。
2. 缺少test oracles以实现自动化测试。

本文基于**蜕变测试**（metamorphic testing）的思想，提出了一种**独立页面模糊测试**（independent view fuzzing）来解决这个问题。其核心思想利用了**页面之间的独立性**，在随机生成的种子测试用例（seed tests，下文简称seeds）上，额外增加一系列独立的事件，从而构建变种测试用例（mutated tests，下文简称mutants）。对于mutants来说，其表现应该与seeds具有一致性，蜕变测试方法通过检验这种一次性来判断软件是否有功能性问题。

所谓页面独立性，指的是对于同一个父组件上平行的组件（如ListView中并排的按钮），或不属于同一个父组件的组件（如ListView的按钮和导航栏的按钮）在功能上应该相互独立。对于其中一个组件的操作不会影响其它组件的正常功能。本文称其为独立页面属性（Independent View Properties）。

给予独立性的概念，mutants和seeds的执行情况应该要保持一致，否则就认定为出现了功能性问题。

本文构建了自动化功能模糊测试工具**GENIE**，在给定一个软件后，GINIE可以做到：

1. 在不需要人为设计测试用例的情况下，自动检测非崩溃bug。
2. 检测到的bug具有多样性和通用性，在多个功能属性上皆有效果。

### 独立页面属性

举例来说明独立页面属性：ActivityDiary应用可以记录每日生活中的照片，这些照片可以被分为多个类别（电影、睡觉、做清洁等）。用户在选择一个类别后，可以上传照片并发布到Diary页。在Dairy页面中，可以对已发布的照片执行删除操作。具体场景如下图描述：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h4zp315khpj220y0f4tde.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h4zp315khpj220y0f4tde.jpg)

初始状态为$l_1$，用户按照以下的流程（记为流程A）操作进行状态转移：

1. 点击Cinema按钮，新增一个Cinema主题的日记，状态转移到$l_2$。
2. 点击相机按钮，选择一张图片后，图片加载到页面上，状态转移至$l_3$。
3. 点击导航栏，返回日记页面，页面显示Cinema标题和图片内容，状态转移至$l_4$。
4. 点击图片，弹出删除对话框，状态转移至$l_5$。
5. 点击确认删除，页面上图片被删除，仅保留Cinema标题，状态转移至$l_6$，流程终止。

从直觉来说，若在删除Cinema的图片之前添加一个Sleeping或Cleaning的日记，对Cinema图片的删除流程是不会有影响的，***这里就体现了页面之间的独立关系***。将上述流程作为seeds，我们可以生成一个mutants：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h50b1u22gpj21z80tc7cr.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h50b1u22gpj21z80tc7cr.jpg)

该流程（记为流程B）表示为：

1. 在$l_3$时，点击Cleaning按钮，新增一个Cleaning主题的日记，状态转移至$u_1$。
2. 点击相机按钮，选择一张图片，图片加载到页面上，状态转移至$u_2$。
3. 点击导航栏，返回日记页面，页面显示Cleaning和Cinema标题和对应的图片内容，状态转移至$l_4'$。
4. 点击Cinema对应的图片，弹出删除对话框，状态转移至$l_5'$。
5. 确认删除，此时可能会出现两种情况：
   1. Cinema图片被成功删除，仅保留Cleaning日记及Cinema标题，状态转移至$l_6'$，流程结束。
   2. Cleaning图片被删除，页面保留Cinema日记内容及Cleaning的标题，状态转移至$u_6$，此时页面间的独立性被破坏，检测到功能问题。

总结来说，正常情况下，在相对一个流程（如流程A）新增一些操作后（如流程B中新建Cleaning日记），原有流程的后续操作（如删除Cinema图片）是不会影响其它组件的（如Cleaning的图片不会被删除）。否则就出现了功能性问题（如Cleaning的图片反而被删除）。

在本例中，独立页面属性体现在：“额外增加一个Cleaning的日记后，对删除Cinema的图片不会造成影响”。这种独立页面属性是广泛存在于安卓软件中的：本文调查了5个有名的、不同类别的且拥有百万下载量的安卓软件，从他们的bug reports中找到129的非崩溃bug，其中有38个（29.5%）体现了独立页面属性，所以该项研究对寻找到这些大量存在的问题是有一定帮助的。

针对以上流程，可以引出了几个关键问题：

1. 我们如何去生成这些seeds？
2. 我们如何生成mutants，以至于可以让额外的事件序列可以体现出页面独立属性？
3. 我们如何去衡量mutants的操作结果和seeds的操作结果存在不一致，即mutants是否影响了正常的

### 问题和主要思路（独立页面属性模糊测试）

为解决上述的问题，本文的主要步骤包括：

1. 构建**图结构**的GUI转移模型（GUI transitional model ），其中图的节点是经过量化的页面的状态，图的边时引发状态转移的操作事件。
2. 根据转移模型，生成一系列seeds。分析seeds中每个页面的布局，以树的形式描绘布局中每个页面间的关系，根据一定的规则找到具有相互独立属性的页面。这些页面往往具有相似的功能，但是在行为上是相互独立的。
3. 利用seeds和转移模型，在seeds中找到一个“支点节点（pivot）”，并且在转移模型中找到一个环路，使其在执行若干个操作事件后可以回到支点节点。这个环路就是mutants中的额外操作事件。
4. 对比seeds和对应mutants是否存在差异，从而判断是否有违背页面独立属性的情况。

![https://tva1.sinaimg.cn/large/e6c9d24ely1h50tk1egkxj21vc0iqq7z.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h50tk1egkxj21vc0iqq7z.jpg)

## 实现方式

### 符号定义

安卓App以GUI为中心，依靠事件驱动GUI的变化。本文将一个App定义为$P$，App中的GUI布局定义为$l$，并用树形结构$T$来表示页面构成情况。树中的每一个节点是一个GUI页面（或组件），记为$w$（于是$w\in l$）。每一个页面$w$拥有一种类型，记为$w.type$，并且：

- 对于叶节点，它的类型可以是按钮（Button）或者文本框（EditText）
- 对于内部节点，它的类型可以是列表页面（ListView）或线形布局页面（LinearLayout）等，我们统称为group view。

如果$P$不在前台运行，则本文定义$l=\bot$。

GUI事件$e=\<t,r,o\>$是一个三元组，其中$e.t$表示事件$e$的类型（如点击、编辑、轻扫）；$e.r$是一个函数，参数为某个布局，返回值为在这个布局上执行了$e$的页面，即$e.r(l)=w$表示在布局$l$中的页面$w$执行事件了$e$；$e.o$表示与事件$e$相关联的数据，这是个可选的属性。

于是一个GUI测试用例可以用一个事件序列$E=[e_1,e_2\dots,e_n]$来表示。在$P$上执行$E$可以得到一个执行过程$\prod_p(E)=\<L,I,H\>$，其中$L=[l_1,l_2,\dots,l_{n+1}]$是过程中变化的页面布局，$I$和$H$分别记录了每个布局中的独立页面和当前活跃的页面（active view）。于是$l_{i+1}=e_i(l_i)$可以表示事件对页面布局的驱动过程。

### 构建GUI转移模型

GUI转移模型用于生成测试用例序列，用$M=\<S,\Sigma,\delta\>$表示。其中$S$是所有抽象状态$s$的集合，且$S$是$L$中布局们的子集（因为不同的布局可能属于同一种状态）；$\Sigma$表示$P$能够接受的所有事件的合集；$\delta :S\times \Sigma \rightarrow S$表示“状态经过事件后，转移到新的状态“这一映射转移关系，每一个转移可以用元组$\<s,e,s'\>\in \delta$来表示。

本文使用了一种C-Lv4的变体来衡量布局$l$和$l'$之间的差异。具体来说，本文将布局$l$表示为它组件的集合$\{w_1, w_2, \dots,w_{|l|}\}$，并且将布局$l$抽象为$l_{abs}=\cup_{w}\in \{w.type\}$，即将页面树结构中的页面用类型来表示，这样做时为了减少状态数量有指数爆炸的威胁。当两个布局的抽象相等时$l_{abs}=l'_{abs}$，认为两个布局相等，记为$l\simeq l'$。

举例来说，当两个布局的结点存在差异时，则判断两个布局不相等，如$l_3$比$l_2$多出了一个imageView结点；而对于ListView这种可以包含零至多个结点的group view，若两个布局只在子节点的个数上有差异，而每个子节点没有差异，则认定为两个布局相等，如$l_4$与$l_4'$中，ListView子节点个数有差异，但每个子节点的类型都是一致的，所以两者相等。

为了能生成更合理的事件序列，本文提出了一种State Coverage-optimized Model Mining算法。对于GUI分布$l$，遍历其所有页面，找到各页面可以执行的事件$e$并执行，从而转移至新的分布$l'$，以此类推从而构成图。在这个过程中，如果找到任意一对$l_{abs}\simeq l'_{abs}$，则$M$中会出现一个环形，这个环形结构在后续的mutants生成中有着重要的作用。其中，对于事件的选择按照以下两个策略实现：

- 系统性事件选择（Systematic event selection）：该策略优先考虑了较少被执行到的，且能够转移到较多新布局的事件。具体地，用$\Upsilon(e)$表示事件$e$的权重，本文为seeds中出现过的所有事件$e$存储一份权重，用于使用和更新：

  $$
  \Upsilon_{i+1}{(e)}=\frac{\Upsilon_{i}(e)+\sum \Upsilon_i(new\_events(e))}{exec\_times(e)^2}
  $$

  其中，$new\_events(\cdot)$表示在$e$执行后的布局$l'$上，曾经执行过的所有事件的集合（即图模型中状态$s$的出度）；$exec\_times(\cdot)$表示事件$e$总共被执行过的次数。初始状态下，$\Upsilon_1(e)=100$、$exec\_times(e)=1$、$new\_events(e)=\emptyset$。

- 随机事件选择（Random event selection）：根据预定义的概率选择事件，其中点击事件概率为60%、长按事件为35%、导航事件为5%。随机事件选择可以减少探索过程中陷入重复选择的情况，如在一个很长的*ListView*中可能点击多个相似的组件。

在具体实现时，首先基于系统性事件选择进行探索，之后交替两种策略构建seeds。这样可以构建出一个图模型：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h54ygie6h2j20yy0u00vs.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h54ygie6h2j20yy0u00vs.jpg)


### 从Seeds中确定独立页面和活跃状态

为了要知道在生成mutants时，哪些组件和事件可以作为起点。以上面的例子来说，即在已知$l_2$与随后的“Cinema点击事件”的情况下，找到与点击Cinema平行且相互独立的“Cleaning点击事件”。

为了做到这点，我们需要知道在seeds中有哪些相互平行、互不干扰的组件（比如他们是同一个列表的两个按钮，或是不同列表的按钮）。同时，我们也应该知道在seeds中有哪些组件已经被事件影响了（比如Cinema已经被点击过了）。

于是本章的重点在于定义“页面相互独立”的规则和记录已经发生时间为的方法。

对于页面相互独立的规则，布局$l$可以被分割为多个不重合的group views $G=\{w_{g_1},w_{g_2},\dots,w_{g_{n}}\}$，其中$w_{g_{i}}\in l$表示布局中一个内部节点。令$\rm{Group}(w)$表示页面$w$所在的group（如按钮$w_1$在ListView $w_2$上，则有$\rm{Group}(w_1)=w_2$）。如果$w_1$和$w_2$两个页面满足以下两个条件：

1. $w_1$和$w_2$属于不同的group，即$\rm{Group}(w_1)\ne \rm{Group}(w_2)$
2. $w_1$和$w_2$属于相同的group，即$\rm{Group}(w_1)=\rm{Group}(w_2)$，且它们的类型也相等，即$w_1.type=w_2.type$

则$w_1\nparallel w_2$。

值得注意的是，若页面不属于任何一个group，则它独立于在其它group的页面。

![https://tva1.sinaimg.cn/large/e6c9d24ely1h534k88d8fj21060j8jva.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h534k88d8fj21060j8jva.jpg)

举例来说，图(d)展示了$l_1$布局的树形结构，其中$w_1$、$w_2$和$w_3$属于同一个group view（$w_0$），且为同类型的兄弟节点，于是它们相互独立。显然，点击其中一个组件，是不会影响到另外两个的；$w_6\nparallel w_1$，因为两者不属于同一个group；$w_4\nparallel w_1$，因为$w_4$不属于任何一个group。

图(e)中展示了$l_4'$布局的树形结构。其中$w_4\nparallel w_1$，因为它们都是$w_0$的子节点。显然点击$w_6$也是不会影响到$w_3$的。

在确定独立规则后，本文还定义了Active View。具体来说，对于布局$l$的group view集合$G=\{w_{g_1},w_{g_2},\dots,w_{g_{n}}\}$，定义一个映射结构$h_l\in H$，记录每个group view的活跃子页面，其规则如下：

1. 对于页面$w_i$，当一个事件$e$关于布局$l$满足$e.r(l)=w_i$时，认为$w_i$是其所在group的活跃页面，即令$h(\rm{Group}(w_i))=w_i$。
2. 对于活跃页面$w_i$，当一个事件$e$关于布局$l$满足$e.r(l)=w_j$，且$\rm{Group}(w_i)=\rm{Group}(w_j)$时，页面$w_i$不再活跃（become inactive），即令$h(\rm{Group}(w_j))=w_j$。
3. 对于页面$w_i$，当一个事件$e$关于布局$l$满足$e.r(l)=w_j$且$\rm{Group}(w_i)\ne \rm{Group}(w_j)$，则$w_i$的活跃情况不受影响，且有$h(\rm{Group}(w_j))=w_j$。

另外，不属于任何一个group的页面被认为是不活跃的。

一个比较重要的点是，在seeds的生成过程中，我们需要记录每一个group view被点击过的页面，这样mutants在探索seeds过程中可以方便地知道每个group中有哪些页面被点击过了。

在更新活跃状态的过程中，GINIE会为每个布局$l$寻找与它最接近的***相似布局***，并在布局中寻找***相似页面***，从而***更新状态***。这里涉及两个问题：

1. 如何界定两个布局、两个页面之间是否相似
2. 如何记录与更新页面的活跃状态

**布局相似性：**对于第一个问题，若两个布局属于同一个主页面，则认为它们相似。本文使用Similar_layout_type($l_1,l_2$)函数反应两个布局是否相似，如示例中的$l_2$和$l_3$虽然页面结构不同，但都属于ActivityDariy这个主页面，所以认为他们是相似的布局。

**页面相似性：**对于页面来说，假设事件$e$作用于布局$l=\{w_1,\dots,w_{|l|}\}$，且$l'=\{w_1,\dots,w_{|l'|}\}$是另一个布局。显然此处有$e.r(l)=w_i$。如果此时存在$w_j'\in l'$，能够满足$e.r(l')=w_j$，即若$w_i$在$l$上可以被事件$e$作用，且$w_j$在$l'$上也可以被事件$e$作用，则认为$w_i$和$w_j$相似，记为$w_i\sim w_j$。

具体来说，为了寻找是否存在$w_j\in l'$和$w_i\in l$相似，本文将页面的类别信息转化为一个字符串，并于所有$w_j\in l'$进行字符串对比，若没找到一致的，则计算字符串编辑距离，找到距离最近的那一个“最相似”的页面。

**活跃状态更新：**对于第二个问题，假设经过事件$e$后的布局为$l$，则从之前的所有布局序列中，按照倒序寻找第一个和$l$相似的布局$l_i$，从对应的$h_i=\{(w_{g_1},w_{a_1}), (w_{g_2},w_{a_2})\dots\}$中，寻找是否存在$w_g'$与$l$的某个group view $w_g$相似，且$w_g'$的活跃页面$w_a'=h[w_g']$与$l$中的某个节点$w_a$相似。若存在这种关系，则为布局$l$的活跃页面关系$h$添加$h[w_g']=w_a'$。

总结来说，独立页面的确定与活跃状态的构建可以用以下流程表示：

1. 初始化空布局列表$L$、空独立页面列表$I$、空活跃状态列表$H$
2. 获取GUI布局$l$，将其加入$L$；获取$l$上的独立页面，加入$I$
3. 获取事件$e$
4. 更新活跃状态$h=[Group(e.r(l))]=e.r(l)$，将$h$加入$H$中
5. 执行事件$e$，获得新的GUI布局$l'$
6. 执行update_active_views($l'$,$\<L,I,H\>$)函数，更新当前的$h$
7. 将$l'$加入$L$，获取$l'$上的独立页面，加入$I$
8. 重复3-8

其中update_active_views函数接收当前的布局$l$，以及历史的布局、独立页面和活跃状态的序列$\<L,I,H\>$。具体步骤如下：

1. 倒序遍历序列$\<l_i,h_i\>\in\<L,H\>$
2. 如果当前布局$l$和$l_i$相似，则：
   1. 遍历每一个$\<w_g,w_a\>\in h_i$ 
   2. 若$l$中存在$w_g'$，有$w_g'\sim w_g$，且存在$w_a'$，有$w_a'\sim w_a$，则令当前的$h[w_g']=w_a'$  

### Mutants生成和执行

Mutants生成的主要思想是，在原有的seeds中，能够找到一个起点（pivot）布局和一个事件序列$\tau$，使得$\tau$的第一个事件可以在起点的布局上执行，而seeds中pivot的后一个事件可以作用在$\tau$的最后一个布局上。同时，需要满足$\tau$的第一个事件作用页面是不活跃的。

其中，对于这些不活跃的页面$b$，一般会存在一个活跃页面$a$与其保持相互独立的关系，即在$b$上的任何操作不会影响$a$的原有操作。

于是本文定义了两个概念：

- 独立事件序列：指的是对于布局$l_i$，存在一个额外的序列$\tau=[e_1',e_2'\dots]$满足$e_1'.r(l_i)$是不活跃的，即独立于$l_i$中任一活跃页面。

- 序列连接：指的是假设两个序列$\tau_1=[e_1,e_2\dots]$和$\tau_2=[e_1',e_2'\dots]$，若$\tau_2$的第一个事件$e_1'$能够在$\tau_1$的最后一个布局$l_{|\tau_1|+1}$上作用，则两个序列相连，记为$\tau_1 \leadsto \tau_2$。

假设seeds的事件序列为$E=[e_1, e_2, e_3,\dots, e_n]$，在生成mutants时，需要在第$i$个位置前插入一个序列$\tau=[e_1',e_2'\dots,e_m']$，生成$E'=[e_1, e_2\dots, e_i, e_1',e_2'\dots e_m', e_{i+1}\dots e_n]$。此时$\tau$应该满足上述的两个条件：

1. 是一个独立事件序列
2. 满足$[e_1\dots, e_{i-1}]\leadsto \tau \leadsto [e_{i},\dots e_n]$ 

举例来说，假设下图是seeds，令$l_3$为pivot布局。$l_3$可以被抽象为状态$s_1$（包含所有页面的类型）。可以看到Cinema、Sleeping和Cleaning是三个相互独立的页面，且属于同一个group view。其中Cinema（记为$w_1$）是活跃的（在$e_1$时有$h[Group(w_1)]=w_1$）。

![https://tva1.sinaimg.cn/large/e6c9d24ely1h4zp315khpj220y0f4tde.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h4zp315khpj220y0f4tde.jpg)

因此，一个合理的事件序列$\tau$可以从点击Sleeping或点击Cleaning开始。根据图模型可以构建下一个事件（点击Camera）。我们以$\tau_1=[Cleaning, Camera]$为例，它能够满足序列连接的条件。首先第一个事件的Cleaning页面在$l_3$上可以找到相似的页面（即Cleaning本身），而过渡到$l_4$的事件（点击Nav）也可以在点击Camera之后的状态$s_1$中执行。

![https://tva1.sinaimg.cn/large/e6c9d24ely1h54ygie6h2j20yy0u00vs.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h54ygie6h2j20yy0u00vs.jpg)

该步骤实现的具体算法如下：

1. 获取seeds的布局、页面和活跃状态序列$\<L,I,H\>$，初始化mutants为空序列
2. 遍历$\<L,I,H\>$中的每一个布局和对应的活跃状态$\<l_i,h_i\>$ 
   1. 获取抽象表示$l_{abs}$ 
   2. 基于Search_traces($i$, $l_{abs}$,$M$)函数获取多个mutants
   3. 将mutants添加至mutants序列
3. 返回mutants序列

其中，Search_traces($i$, $l_{abs}$,$M$)函数的具体步骤为：

1. 初始化候选序列为空，初始化空栈
2. 基于图模型$M$，遍历$l_{abs}$的所有出度关系，即$\<l_{abs}, e, l_{next}\>$ 
   1. 如果事件$e$可以作用在$l_i$上，且作用的页面是不活跃的，则将$[e]$推入栈中，此处$e$为只有一个事件的事件序列
3. 当栈不为空时，有：
   1. 获取栈中的第一个事件序列$\tau$ 
   2. 获取$\tau$的前一个布局$l_{\tau-last}$ 
   3. 若$(l_{abs}\simeq l_{\tau-last})\wedge ([e_1\dots, e_{i-1}]\leadsto \tau \leadsto [e_{i},\dots e_n]) \wedge (|\tau|< \rm{MAX\_LENGTH})$，则将$\tau$加入候选序列；否则遍历$M$中$l_{\tau-last}$的出度，将其拼接在$\tau$后并且加入栈中
   4. 当候选序列超出预期个数时返回所有候选序列

在3.a.中，MAX_LENGTH表示插入序列最长的长度，此步骤也反应了算法广度优先搜索的策略，若当前找到的序列不满足连接关系，则在序列中再添加一个步骤查看是否符合条件。

### 基于GUI的oracle checking

在检查seeds和mutants的差异时，主要遵循的思想是：对于mutants中额外的操作，相对seeds来说只会在UI上相对增加GUI效果，而不会减少GUI效果。

本文对GUI效果定义可以概括为：为两个类似布局（即属于同一个父页面的布局，如同为DiaryActivity）的GUI变化。具体的定义如下：

**GUI效果：**对于布局列表$L$和对应的事件序列$E$，在给定两个相似布局$l_i$和$l_j$且$i<j$的情况下，他们的GUI效果被描绘为$l_i\backslash l_j$与$l_j\backslash l_i$之间去重后的并集，这里的”$\backslash$”符号定义为两个布局之间的差，即$l_i$中存在而$l_j$中不存在的页面。最终GUI效果被记为$\Delta(l_i, l_j)=(l_i\backslash l_j)\uplus (l_j\backslash l_i)$。

$\Delta(l_i,l_j)$有三种情况：

1. $(w,\perp)\in \Delta(l_i,l_j)$表示$l_i$中的$w$被删除了
2. $(\perp, w)\in \Delta(l_i,l_j)$表示$l_j$中新增了$w$
3. $(w,w')\in \Delta(l_i,l_j),w\ne w'$表示$w$被删除，且新增了$w'$ 

可以看到，若在seeds中存在$\Delta(l_i,l_j)$，与其在mutant中对应的$\Delta(l_i',l_j')$有着以下关系：$\Delta(l_i,l_j)\not\subset \Delta(l_i',l_j')$，则存在GUI不一致的问题。

为了增加算法性能，我们其实只需要考虑那些会被mutants影响的、且相似的布局，观察其是否存在GUI不一致的现象。具体的算法如下：

1. 获取seeds布局列表$L=[l_1,\dots,l_k,l_{k+1},\dots l_{n+1}]$和mutants的布局列表$L'=[l_1,\dots,l_k]::L_{\tau}::[l_{k+1}',\dots,l_{n+1}']$，初始化测试通过flag为True
2. 对于$L$的每对布局$(l_i,l_j)\in L\times L$，筛选出$(i\le l) \wedge (k+1 \le j)$或者$k+1\le i \le j$的元组
3. 判断改元组是否为相似布局，若是，则执行4，否则继续遍历
4. 判断是否满足$\Delta(l_i,l_j)\not\subset \Delta(l_i',l_j')$，若是，则flag置为False，否则继续遍历

## 实现细节

### Seeds生成

在seeds生成时，我们应该尽可能地让测试用例有多样性、实用性和可扩展性。本文在生成用例的过程中保持和更新选取各event的权重来增加测试覆盖率，同时也会选取一些更有意义的操作（比如对话框中选择“是”，而不是“取消”）

### Mutants生成

本文为了提升mutants生成的效率，做出了以下三点工作：

1. 在生成mutants时本文使用了广度优先搜索，经过研究这比深度优先搜索更具有效率。
2. 同时为了避免路径爆炸的问题，在序列选择时，每个环路最多被选择两次，且每个group只考虑三个独立页面。
3. 图模型删除了两个state间冗余的事件，来缩小搜索范围。对于一些不能保证复现的用例，本文保存了这些用例的前缀信息，在探索到包含这些前缀信息的用例时进行剪枝。

### 减少重复错误和假阳性样本

GINIE报告的错误中，有许多是重复的错误，或是假阳性样本（实际上不是错误，但被判定为错误）。

为了减少重复性错误，本文将所有疑似功能错误转化为字符串，字符串中保留了oracle violation的信息，通过比较字符串的方式去除重复的错误，同时也统计每个相同字符串出现的次数。本文认为，在这些重复出现的字符串中，出现次数较少的更可能是真实的问题（我们乐观地认为真实的bug不会出现太频繁）。具体来说，本文只选择了出现一次的字符串对应的错误。（当然，这会导致模型漏判）

对于假阳性问题，很多都是由于软件的一些特性以及对布局的抽象导致的。为了解决这个问题，本文使用了以下两个策略：

1. 对于每个seed执行两次，来找到那些每次执行情况不一样的页面（如对时间敏感的、或是有随机行为的页面），在oracle checking时不考虑这些页面。
2. 利用启发式算法来判断一个事件序列是否能够回到pivot页面：若序列最后一个页面与pivot页面的布局相似度小于50%，则认为它没有回到pivot，并抛弃这条mutant。

## 评估

本文在评估GINIE时，专注于以下几个问题：

- RQ1（Bug Finding）：GINIE能否在真实软件中找到这些非崩溃bug？现有的软件测试工具能否做到这些？
- RQ2（Code Coverage）：GINIE能否通过模糊测试提升代码覆盖率？
- RQ3（Oracle Precision）：GIENE对检测非崩溃功能性bug的准确率有多少？
- RQ4（Bug Types）：GINIE主要能够找到什么类型的非崩溃bug？

### RQ1:Bug Findings

![https://tva1.sinaimg.cn/large/e6c9d24ely1h563c4yvb3j21100m4wjk.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h563c4yvb3j21100m4wjk.jpg)

表3-表5展示了GINIE在26个软件中找到的bug情况统计。GINIE找到了34个非崩溃bug和26个崩溃bug（共60个），其中58个被证实，36个被修复。对比来说，MONKEY和APE对没有找到任何一个非崩溃bug。

表5详细展示了对于每个软件检测的详细情况，分别对应：

- #Generated Mutants：生成的mutants个数
- #Executed Mutants：执行的mutants个数
- #Error Mutants：发现疑似错误的个数
- #D.E. Mutants：去重后的疑似错误个数
- #1-O.D.E. Mutants：仅出现一次的疑似错误个数
- #1-O.D.E. Mutants(Refined)：减少假阳性样本后的错误个数
- #T.P. Mutants：真实bug个数
- #Distinct Non-crashing Bugs：去重后的非崩溃Bug数
- #Distinct Crash-Bugs：去重后的崩溃Bug数

### RQ2:Code Coverage

![https://tva1.sinaimg.cn/large/e6c9d24ely1h5642ef5jmj210s0j2tdt.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h5642ef5jmj210s0j2tdt.jpg)

表6展示了各软件所有seeds和mutants的的代码覆盖情况，包括行数占比、分支数占比和方法数占比。此处使用了Jacoco来衡量代码的覆盖率。可以看到mutants在各指标的覆盖率上都有所提升。

### RQ3:Oracle Precision

从表5可知，GINIE产生了许多假阳性样本，具体统计情况如下所示：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h567rix80rj20hk0k4q4p.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h567rix80rj20hk0k4q4p.jpg)

本文把引起假阳性的原因分为三类：

- 页面独立属性被破坏（记为#I）
- 布局类型推断不准确（记为#L）
- 状态抽象不够准确（记为#S）

![https://tva1.sinaimg.cn/large/e6c9d24ely1h5698f87xqj20u00y4wj6.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h5698f87xqj20u00y4wj6.jpg)

**页面独立属性被破坏：**GINIE所选择的组件虽然和其余组件不属于一个group或是同一个group中的兄弟关系，它们在GINIE中被定义为相互独立，而实际上有依赖关系。如图4第一列中，UnitConverter中的刷新按钮和其余按钮不在同一个group，却会影响页面中From和To两列的状态；AnyMemo中的随机按钮和重复按钮是同一个group的兄弟关系，但它们同时开启时会相互影响。

**布局类型推断不准确：**在oracle checking时，GINIE会选择类型相似的布局进行比较并计算GUI effects，评判“相似”的标准是属于同一个主页面。但有时同一个主页面的两个布局功能差异较大，GINIE很难自动判断，如图4第二列中的FEATURED与BOOKMARKS是完全不同的两个功能，但是会被GINIE判断为相似的布局。

**状态抽象不够准确：**在构建mutants时，$\tau$的结尾需要和pivot达成闭环，闭环的条件是结尾的布局要和pivot相似，相似的条件是其页面的类型相等。但有的页面仅仅是在title等属性上不一致，但是确实两个不同功能的页面，此时GINIE会把它们当作相似的页面，从而构建了不合适的mutant。如图4第三列中的“New Activity”页面的“Gardening”页面。

尽管GINIE的准确率只有40.9%（260/635），这也比当前较为著名的工具（如Facebook的INFER的26.3%）要高出不少，且很多bug都是开发者编写测试用例也无法测试出来的。另外GINIE并不需要投入太多人力去找到这些bug，只需要在找到的bug中筛选出真实的bug，很大程度上降低了时间成本。

### RQ4:Bug types

本文总结了6种类型的bug，并举例说明：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h56b6w9rrmj21ez0u0n46.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h56b6w9rrmj21ez0u0n46.jpg)

1. 数据丢失：一些bug会导致用户数据的丢失，如Markor是一个基于markdown的笔记工具，如果用户新建了一个和已有文件相同标题的笔记，则新的文件内容会覆盖旧的文件内容。（图5列1）
2. 函数不被执行：一些bug导致某些功能突然失去作用，如SkyTube是一款播放youtube视频的工具，当用户在菜单中选择“在浏览器中播放”后返回软件，则用户无法点击开始或暂停按钮（图5列2）
3. 错误行为：一些特定的功能会有异常的行为，比如错误的页面被打开，或是某个已经被删除的页面被打开。如ActivityDiary中（图5列3）：
   1. 点进一个日记（如Officework）中
   2. 将其名字修改为Shopping
   3. 重新建一个名为Gardening的日记，在不保存的情况下返回
   4. 打开Shopping时，出现的是Gardening的页面，若此时修改内容会引起数据不一致问题
4. GUI状态不一致：页面会出现与功能描述不一致的情况，如音频播放器Transistor（图5列4），在音频播放时，音频文件名旁会出现绿色标识，表示正在播放；当音频暂停时标识会消失。但当在播放时对文件重命名，再暂停，则播放标识不会消失。
5. GUI页面展示错误：一些页面存在重复展示的问题。如Task中（图5列5），当创建一个包含标签（如Meeting）的新任务后，若再额外插入两个附件，则页面上的会出现两个相同的标签；一些页面存在组件突然消失的问题，如Fosdem中（图5列6），在连续点击两次主菜单中的“搜索事件”按钮后，主菜单左边的按钮会消失。
6. GUI设计问题：这种类型的bug和功能性GUI涉及问题相关，这容易影响用户体验。比如RadioDroid的历史播放页面遗漏了一些音频，原因是程序保存了两种类型的历史数据“history”和“track history”。其中track history只记录了音轨信息。但程序并没有清晰的区分这两种数据。

## 讨论

### GINIE对长期存在问题的解决

本文对GINIE检测到的34个非崩溃bug进行分析，结果如下图所示：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h56gl0c5drj20vg0suacq.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h56gl0c5drj20vg0suacq.jpg)

78.8%（26/34）的bug超过一年没有被解决，57.6%（19/34）的bug影响了超过10个版本。

### seeds和mutants的质量

下表描绘了seeds和mutants的质量：

![https://tva1.sinaimg.cn/large/e6c9d24ely1h56gn7pndfj20ji0fuq4r.jpg](https://tva1.sinaimg.cn/large/e6c9d24ely1h56gn7pndfj20ji0fuq4r.jpg)

其中B/A表示能够生成错误的seeds的比例，T.P. Insert.Pos.表示mutant插入位置的正确率。

### 效度威胁

本文主要有两个主要的效度威胁：

1. 本文只评估了12个软件。为了解决这个问题，本文选择了不同类型的软件，来保证评估的泛化性。
2. 假阳性的分析是人为开展的，本文通过多个共同作者的交叉验证来解决这个问题。

## 未来的工作方向

在没有oracles的情况下进行安卓软件的自动化功能验证，是一个很具挑战性的问题。

1. 页面独立性虽然可以找到很多类型的bug，但是尚不能覆盖所有类型的功能性问题。目前GINIE只能通过GUI的差异找到那些会引发GUI不一致的问题。
2. 目前GINIE只利用了随机生成的seeds来检测bug，后续可以使用一些人为提供的测试用例来检测更多的功能问题。