+++
title = 'Multimodal Biometric Feature Learning Notes'
date = 2021-06-10 18:51:25
math = true
+++

生物识别可以安全快速地鉴别个体，具体可以通过指关节纹（Finger Knuckle Print，FKP）、指纹（Finger Print)、掌纹（Palm Print）等。由于疫情影响，FKP识别成为一个不易传播疾病的识别方式。

# LEARNING DISCRIMINATIVE FINGER-KNUCKLE-PRINT DESCRIPTOR

![](https://hagemon.github.io/post-images/1623326898334.png)
（ICASSP 2019）方向信息(Direction information)是识别FKP图像的一个关键特征，大多现有的方法仍然使用这人工设计的特征，这些特征需要很多的先验知识。这篇文章为FKP的特征学习，提出了一种基于方向的二值特征学习方法(Discriminative Direction Binary Feature Learning, DDBFL)。主要贡献有：
1. 设计了一种基于Gabor过滤器卷积的特征Direction Convolution Difference Vector(DCDV)
2. 使用投影函数将特征二值化，并保持同类样本的紧密性(compact)和异类样本的差异性(separable)
3. 将特征进行分块统计，组成最终的FKP特征算子(Feature Descriptor)

### Gabor Filter

Gabor滤波器使用了图像频域中的信息来提取图像的纹理特征，由一个二维高斯函数和二维三角函数叠加而成，本文使用了Gabor滤波器的实数部分

$$
G(x,y,\theta,\sigma, \beta)=\frac{1}{2\pi \sigma\beta}\exp\{-\pi(\frac{x'^2}{\sigma^2}+\frac{y'^2}{\beta^2})\}\cos({2\pi \mu x'}) 
$$

其中

$$
\begin{aligned}
x'=(x-x_0)\cos\theta+(y-y_0)\sin\theta \\\\
y'=-(x-x_0)\sin \theta + (y-y_0)\cos \theta
\end{aligned}
$$

$(x_0,y_0)$为核的中心，$\mu$是径向频率（我取1/3），$\sigma$和$\beta$为$x,y$标准差，这里取1。

### DCDV
在本文中，当选定了一个方向$\theta$后，根据核的大小便可以计算出每个方向的Gabor核，并在图像上进行卷积操作，得到每一个像素在该方向的特征值。本文从$[0,\pi]$选择了12个方向，为每个像素计算了12个特征值，将12个特征值$c_1,c_2,\dots,c_{12}$的相邻差组成向量，即DCDV：
$$DCDV=[c_{12}-c_{11},c_{11}-c_{10},\dots,c_2-c_1,c_1-c_{12}]$$
![](https://hagemon.github.io/post-images/1623333562583.png)
DCDV可以很好的表示图像中出现的多个主要方向特征，若相邻的两个卷积结果异号，说明此处包含着方向信息。同时，DCVC的均值为零，所以不需要做归一化。

### DDBFL

特征的二值表示是一个具有鲁棒性的表示方法，若原始特征值具有一定动荡性，它的二值表示还是能相对稳定一些的，如$[0.8,0.9]$的动荡，在二值表示后还是会归为$1$。本文使用一个投影矩阵$W\in R^{k\times d}$将$d$维的DCVC投影至$k$维的DDBFL，具体计算如下：
$$b_{p,i,k}=\frac{1}{2}(sgn(w_k^T x_{p,i})+1)$$
表示第i个样本第p个像素的第k维二值特征。

### Objective

优化目标主要从两个角度：
1. 增加特征的差异性，使特征之间长得不一样，更具辨识度，即特征的方差最大化。
2. 使同类样本接近，异类样本远离
于是可以将优化目标初步表示为：

$$
\begin{aligned}
\max_{w_k}J(w_k)&=\max_{w_k}J_1(w_k)+2\lambda J_2(w_k) \\\\
&=\max_{w_k}\sum_{p=1}^P \sum_{i=1}^N ||b_{p,i,k}-\mu_{p,k}||^2 \\\\
&+2\lambda\sum_{p=1}^P\sum_{i=1}^N (\sum_{j\in \Gamma(i)}||b_{p,i,k}-b_{p,j,k}||^2 -\sum_{j\notin \Gamma(i)}||b_{p,i,k}-b_{p,j,k}||^2 )
\end{aligned}
$$

#### 方差最大化

由于上式的sgn函数导致了NP难的问题，所以实际优化过程中做了相应的放宽，直接优化投影后的特征（添加非线性函数？），使投影后的特征与投影后的特征均值平方差最大（即方差最大）：

$$
\begin{aligned}
J_1(W)&=\sum_{p=1}^P |||W^TX_p-W^TM_p||^2 \\\\
&=tr(W^T(\sum_{p=1}^P(X_pX_p^T-2X_pM_p^T+M_pM_p^T))W)
\end{aligned}
$$

这里将矩阵的二范数转化为矩阵的迹形式方便计算，其中$M_p$为所有像素p上的DCDV均值。**这里的均值是每一张图像在像素p上的向量均值，如果每张图像的拍摄范围有一定的偏差，说不定会对优化造成影响**。该方法是否能具有平移不变性的特征还需要进一步实验证明，可以对训练图像进行随机程度的偏移，观察是否会对模型带来影响。

#### Similarity Preserve

在实际计算中，本文把二值特征的欧氏距离改写成内积形式，能这么做是由于二值特征的取值特性$b\in {-1,1}$，即把欧氏距离变成汉明距离。

$$J_2(W)=\sum_{p=1}^P\sum_{i=1}^N (\sum_{j\in \Gamma(i)}sgn(w_k^Tx_{p,i})\times sgn(w_k^Tx_{p,j})-\sum_{j\notin \Gamma(i)}sgn(w_k^Tx_{p,i})\times sgn(w_k^Tx_{p,j})))$$

此处$\Gamma(i)$表示与样本$i$同类别的样本。**可以看到，对于不同的样本来说，也是计算对应的像素之间的差异，再使其差异最小（同类情况下）或最大（异类情况下），也需要验证是否由偏移不变性。**

整理成矩阵形式为

$$
\begin{aligned}
J_2(W)&=\frac{1}{2}\sum_{p=1}^Ptr(W^TX_pSX_p^TW) \\\\
&=\frac{1}{2}tr(W^T(\sum_{p=1}^PX_pSX_p^T)W)
\end{aligned}
$$

其中矩阵连乘的形式相当于样本之间的笛卡尔积，$S$代表相似度。

#### Optimization

用平衡因子结合两个优化项：

$$
\begin{aligned}
J(W)&=J_1(W)+2\lambda J_2(W) \\\\
&=tr(W^T(\sum_{p=1}^P(X_pX_p^T-2X_pM_p^T+M_pM_p^T))W) \\\\
&+\lambda tr(W^T(\sum_{p=1}^PX_pSX_p^T)W) \\\\
&=tr(W^TQW)
\end{aligned}
$$

将中间的连乘矩阵组合为$Q$，在优化时计算矩阵Q最大的k个特征向量，作为该问题的解。

### DDBFL特征表示

以上的步骤将每一个像素的d维特征转化为k维binary code，每张图片的特征维为$w\times h \times k$，其中w和h为图片的宽高。本文进一步将图像划分为不重叠的若干个区域（如$16\times 16$），计算每一个区域中取值的统计值，将图像表示为$16\times 16$的统计特征，计算两个图像之间的卡方距离作为相似度度量。

在我个人的理解里，每个区域内需要统计k个维度中1的个数，这样每个区域相当于有k个变量，再计算所有区域这k个变量的均值，用于计算卡方距离。对于两个不同的图片中的某个区域，它们的卡方距离为：

$$

\chi^2(a,b) = \sqrt{(\sum_{i=1}^k\frac{a_i-m_i}{m_i})^2+(\sum_{i=1}^k\frac{a_i-m_i}{m_i})^2}

$$

其中$a_i$为图片$a$的第$i$维统计量，$m_i$为所有图片在该区域统计量中第$i$维的均值。

### 思考

本文利用了Gabor Filter卷积获得图像的方向特征，为FKP图像提供较具区分度的特征表示。本文的思考如下：
1. 在优化过程中计算两个像素同一个像素的差异，然而图像特征在位置上的分布可能会不同，再加上图像的偏移也可能会影响模型结果，所以可以考虑设计一些能够具有平移不变形的特征提取方法；虽然最后的DDBFL表示可以在一定程度上缓解图像平移的影响（因为是取了一个local patch的统计信息），但这只是在匹配时考虑这个问题，而不是在优化中考虑的；
2. DDBFL的统计表示维度依然很大，可以考虑设计一个维度更小，但是保留更有用信息的表示方法；
3. 是否可以增强更具方向性的特征，平滑不明显的特征，考虑在DCDV上做适当的处理。

接下来需要实现一下这篇论文的模型，并且测试图像偏移的影响。

# Jointly learning compact multi-view hash codes for few-shot FKP recognition

(2021 Pattern Recognition)本文针对FKP图像稀少的情况提出了解决方案，并在DCDV的基础上，利用了多视图特征，从不同的角度来描述FKP图像(multi-view feature)。

### Related Works

本文介绍了一些现有的方法，主要包括hand-crafted特征的方法：
1. Morales et al. 使用尺度不变的特征变换(scale invariant feature transform)来表示FKP.
2. Kumar et al. 基于localized Radon变换(localized Radon transform)对随机线条和褶皱进行编码.
3. 许多现有的方法模仿掌纹识别，使用了基于方向的特征来表示FKP.
4. Zhang et al.基于Gabor过滤器来找到图像的主要方向特征(dominant direction feature).
5. Zhang et al.进一步改进自己的方法，过滤了一些plane pixel，并对特征进行量化(quantify)
6. Gao et al.使用多个方向的Gabor过滤器来获取多方向的特征
和基于学习的方法：
1. 基于PCA和LDA进行特征提取
2. 基于CNN和基本的增强方法(BN, data augmentation)
3. 基于hash learning的方法
4. 基于稀疏表示的学习
这些方法都需要大量的标记数据，对few-shot的情况可能会出现优化不足的情况。
本文的主要贡献为：
1. 提取了多视图的FKP特征(multi-view feature)，提供了更充分的信息；
2. 提出了一种非监督的方法，用于特征学习(feature learning)和特征编码(feature encoding)，同时该方法可以拓展为更多维度的特征学习；

## Learning compact multi-view FKP descriptor

多视图特征指的是用不同类型的特征来对某个图像进行表示，本文主要使用了和DCVC相似的，基于方向的DVDV(Direction View Data Vector)，与基于纹理的TVDV(Texture View Data Vector)

### DVDV

DVDV依旧使用了12个方向的Gabor Filter来提取特征向量，并在之后的步骤中被映射为更低维度的特征向量。

$$DCDV=[c_{12}-c_{11},c_{11}-c_{10},\dots,c_2-c_1,c_1-c_{12}]$$

### TVDV

现有的基于纹理的方法大多使用LBP和PDV来描述特征，它们主要通过相邻像素直接的计算得到特征，当存在一些噪声像素(noise pixel)和描绘的偏差(illumination variances)时会很容易受影响。

本文则在局部区域计算像素的梯度来表示纹理特征。具体来说，使用了8个Kirsch masks来计算每个3x3区域内，八个方向的像素之间的变化梯度。在计算过程中需要把FKP图像缩放到[0,1]之内，以至于和DVDV的取值范围相同。

对于每个像素，可以计算8个梯度值，则每个TVDV可以表示为

$$TVDV=[e_8,e_7,e_6,\dots, e_1]$$

![](https://hagemon.github.io/post-images/1623931007143.png)

### JLCMHC

DVDV和TVDV共同构成了MVDV(Multi-View Data Vector)，为了进一步进行特征学习，本文分别将两个向量映射为K维二值向量：

$$b_{p,i}^v=0.5\times (sgn((W^v)^Tx_{p,i}^v)+1^{K\times 1})$$

其中$b_{p,i}^v$是第v中view的特征（这里是DVDV或者TVDV）。

最终两个K维二值向量将组合成一个K维二值向量$c$作为该图片的哈希表示，向量$c$将通过学习的形式获得。

损失函数包括三个部分：
1. 待学习的向量$c$与投影后两个维度的特征差距最小化
2. 二值化后的特征之间的方差最大化，使得向量之间具有差异性，分布更加分散，每个图像的特征就更加鲜明，不容易被识别为别的特征
3. 使两个view的二值特征差异最大化，这样不同的view之间可以拥有不同的特征

本文使用的是无监督的方法，应对数据集和标记较少的情况。

损失函数的形式为：

$$
\begin{aligned}
\min_{W^v}J(W^v)&=J_1(W^v)+\lambda_1J_2(W^v)+\lambda_2J_3(W^v) \\\\
&=\sum_{v=1}^{V=2}(\beta^v)^r(\sum_{p=1}^P\sum_{i=1}^N ||c_{p,i}-0.5^{K\times 1}-(W^v)^Tx^v_{p,i}||^2 \\\\
&-\lambda_1\sum_{p=1}^P\sum_{i=1}^N||b_{p,i}^v-m_p^v||^2) \\\\ 
&-\lambda_2\sum_{p=1}^P\sum_{i=1}^N||b_{p,i}^1-b_{p,i}^2||^2 \\\\ 
s.t. c_{p,i}={0,1}^{K\times 1}, (W^v)^TW^v=I
\end{aligned}
$$

其中第一项使得学习到的联合向量表示$c$分别与两个投影后的K维特征距离相近，表示将多个view的特征进行融合；第二项损失函数对于所有图像中的每一个像素，其特征方差最大化，具体来说，本文选取了$55\times 110$大小的图像，对于每一个像素位置$(x,y), x\in[0,54], y\in[0,109]$，计算每一张图像在$(x,y)$上特征的方差，并使该方差最大化；第三项使得不同view的每个特征向量差异最大化，使得不同view的特征具有差异性。

在限制条件中，将$c$限制为了二值向量，同时也限制了$(W^v)^TW^v=I$，后者的目的是为了让投影矩阵的每一行都线性独立（与其它行的内积为0），且每一行都是单位向量（与自己的内积为1），这样能够提取更丰富、更discriminative的特征。

可以看出对于第二项损失，可能依然存在对图像平移敏感的情况，需要数据集有一个较为规整的预处理，把图像中心都统一到一个特定的位置，而一三项实际上是存在一定的矛盾的（即要让不同view的特征差异大，又要让最终的联合向量与不同view的投影结果相近），实际上是在找两者之间的一个平衡，可以考虑是否有更合适的向量表示方法。

### Optimization

将损失函数写成矩阵形式：

$$
\begin{aligned}
\min_{W^v}J(W^v)&=\sum_{v=1}^{V=2}(\beta^v)^r(\sum_{p=1}^P ||C_p-0.5-(W^v)^TX^v_{p}||^2 \\\\
&-\lambda_1\sum_{p=1}^P||B_{p}^v-M_p^v||^2) \\\\ 
&-\lambda_2\sum_{p=1}^P||B_{p}^1-B_{p}^2||^2 \\\\ 
&s.t. C_{p}={0,1}^{K\times N}, (W^v)^TW^v=I
\end{aligned}
$$

因为该损失函数中含有不可微函数$sgn(\cdot)$，所以是非凸的，这里将它relax成投影形式：

$$
\begin{aligned}
\min_{W^v}J(W^v)&=\sum_{v=1}^{V=2}(\beta^v)^r(\sum_{p=1}^P ||C_p-0.5-(W^v)^TX^v_{p}||^2 \\\\
&-\lambda_1\sum_{p=1}^P||(W^v)^TX^v_{p}|-M_p^v||^2) \\\\ 
&-\lambda_2\sum_{p=1}^P||(W^1)^TX^1_{p}|-(W^2)^TX^2_{p}|||^2 \\\\ 
&s.t. C_{p}={0,1}^{K\times N}, (W^v)^TW^v=I
\end{aligned}
$$

这里的relaxtion遵循了2010年的论文Semi-supervised hashing for scalable image retrieval，该文的描述为“直观地让同类特征在保有相同正负符号的同时尽可能的拉近，让异类特征在正负符号不同的同时尽可能的远离”。

优化过程分为三步：
1. 更新像素p的联合向量$C_p$
2. 更新不同view的投影矩阵$W^v$
3. 更新权重参数$\beta$

#### 更新$C_p$

$$
\begin{aligned}
 J(C_p)&=\sum_{p=1}^P\sum_{v=1}^{V=2}(\beta^v)^r( ||C_p-0.5-(W^v)^TX^v_{p}||^2) \\\\
 &= \sum_{p=1}^P\sum_{v=1}^{V=2}(\beta^v)^rtr((C_p-0.5-(W^v)^TX^v_{p}|)C_p-0.5-(W^v)^TX^v_{p}|^T) \\\\ 
  &= \sum_{p=1}^P\sum_{v=1}^{V=2}(\beta^v)^rtr(C_pC_p^T-2(0.5+(W^v)^TX_p^v)C_p^T \\\\
  &+(0.5+(W^v)^TX_p^v)(0.5+(W^v)^TX_p^v)^T) \\\\ 

&s.t. C_{p}={0,1}^{K\times N}
\end{aligned}
$$

令其导数为0，可得：

$$
\frac{\partial J}{\partial C_p} = \sum_{p=1}^P\sum_{v=1}^{V=2}(\beta^v)^r (C_p-0.5-(W^v)^TX_p^v)=0
$$

整理可得：

$$
C_p=0.5+\frac{1}{\sum_{v=1}^V(\beta^v)^r}\sum_{v=1}^V(\beta^v)(W^v)^TX_p^v \\\\ 
s.t. C_p=\{0,1\}^{K\times N}
$$

为了保证结果能够符合限制条件$s.t. C_p=\{0,1\}^{K\times N}$，本文将该式调整成符合限制的形式：

$$
\begin{aligned}
C_p&=0.5+0.5\times sgn(\frac{1}{\sum_{v=1}^V(\beta^v)^r}\sum_{v=1}^V(\beta^v)(W^v)^TX_p^v )\\\\ 
C_p&=0.5\times sgn(\sum_{v=1}^V(\beta^v)(W^v)^TX_p^v +1)
\end{aligned}
$$

其中由于$\beta$是正值，不影响$sgn$后的结果，所以可以剔除。

#### 更新$W^v$

本文在这里将损失函数中l2-norm形式改写成矩阵的迹的形式：

$$
\begin{aligned}
J(W^v)=& \sum_{v=1}^V(\beta^v)^r (\sum_{p=1}^P tr( (C_p-0.5)(C_p-0.5)^T \\\\
& -2(C_p-0.5)(X_p^v)^TW^v+(W^v)^TX_p^v(X_p^v)^TW^v) \\\\
&- \lambda_1\sum_{p=1}^P tr((W^v)^TX_p^v(X_p^v)^TW^v-2(W^v)^TX_p^v(\bar{M}_p^v)^TW^v\\\\
&+(W^v)^T\bar{M}_p^v(\bar{M}_p^v)^TW^v))\\\\
&-\lambda_2 \sum_{p=1}^P tr((W^v)^TX_p^1(X_p^1)^TW^1-2(W^1)^TX_p^1(X_p^2)^TW^2\\\\
&+(W^1)^TX_p^1(X_p^2)^TW^2)
\end{aligned}
$$

删除常数项$tr( (C_p-0.5)(C_p-0.5)^T$后可整理为：

$J=W^TYW-2W^TZ$

的形式，这是一个经典的QPSM(Quantitative Strategic Planning Matrix)问题，可以用GPI算法解决,参考F. Nie, R. Zhang, X. Li, A generalized power iteration method for solving
quadratic problem on the Stiefel manifold, Sci. China Inf. Sci. 60 (11) (2017)
112101.。


#### 更新$\beta^v$

$beta$的更新可以整理成带约束优化问题：

$$
\begin{aligned}
&J(\beta^v)=\sum_{v=1}^V(\beta^v)^rg^v \\\\
&s.t. \sum_{v=1}^V \beta^v = 1
\end{aligned}
$$

利用拉格朗日乘子法可以求解得出：

$$\beta^v=\frac{(g^v)^{1/(1-r)}}{\sum_{v=1}^{N_v} (g^v)^{1/(1-r)}}$$

### LCMHC-based FKP representation

与DDBFL特征表示一致，本文把每张图片划分为不重叠的若干个区域，在每个区域进行特征的直方图统计，最后对比每个区域之间的chi-square距离，其实这么做在一定程度上也减少了图像平移所带来的影响。





