---
title: RetinaNet

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-10-10"
tags: ["object detection"]
---

# Focal Loss for Dense Object Detection 

one-stage的检测器比two-stage的更快更简单，但是在准确度上落后。本文发现最主要的原因在于极度的foreground与background的比例失衡（class imbalance）。由此，本文对标准的cross entropy进行修改，并提出Focal Loss。这种loss function可以减小easy examples占的整体loss的比重。这样就使得模型在hard examples下训练，从而减小大量easy examples对训练的影响，解决class imbalance的问题。此外，本文还提出RetinaNet。

## R-CNN这种two-stage的检测器如何解决class imbalance？

- a two-stage cascade。像R-CNN这种two-stage的模型先region proposal，然后再对proposal的结果进行分类。这中间形成了一种级联，使得在第一步region proposal的过程（如RPN，selective search）中就筛除了大量的easy examples（这种过程也可视为一种sampling heuristics）。

- 在第二阶段分类的时候，也使用了sampling heuristics（如fixed foreground-background ratio 1:3 和online hard example mining）。

而one-stage detector却需要处理更多的candidate object locations，通常会有～100k个candidate locations。尽管one-stage的检测器也可以使用上述第2点的sampling heuristics，但是由于基数过大，还是会造成训练过程被easy examples主导，由此导致低效以及模型的泛化能力降低。

## Focal Loss

对于二分类而言，cross entropy表达式如下：

$$CE(p,y)=\left\\{ \begin{aligned} & -log(p) & & if \  y=1 \\\ & -log(1-p) & & otherwise \end{aligned}\right.$$

其中p表示预测到y=1的那一类的概率。为了方便，定义$p_t$:

$$p_t=\left\\{\begin{aligned} & p & & if \ y=1 \\\ 
& 1-p & & otherwise \end{aligned}\right.$$

### Balanced Cross Entropy

Balanced Cross Entropy对标准cross entropy进行了改造：

$CE(p_t)=-\alpha_tlog(p_t)$

其中当y=1时，$\alpha_t=\alpha$，而当y=-1时，$\alpha_t=1-\alpha$。

这样做之后，可以通过调节$\alpha$的值来调节positive-negtive samples分别对loss的贡献的比例。但是这样并不能区分hard examples和easy examples对loss的贡献。所以引入了Focal Loss。

### Focal Loss 

$FL(p_t)=-(1-p_t)^\gamma log(p_t)$

Focal Loss在cross entropy的基础上添加了一个权重$(1-p_t)^\gamma$。这个权重可用于调整不同样本对loss的贡献。对于y=-1而言，一个easy example的预测结果将会使$p_t$增大，从而使得其权重$(1-p_t)^\gamma$减小，所以easy example的权重会减小。

在实践中，Focal loss通常会与balanced cross entropy结合起来:

 $FL(p_t)=-\alpha_t(1-p_t)^\gamma log(p_t)$

### Class Imbalance and Model Initialization

二分类通常会默认将模型初始化使得起初分类到y=1和y=-1的概率相等。但是在class imbalance的情况下，将两类的初始预测值设为相同会使得frequent class的loss主导整个loss，使得训练不稳定。于是，本文设计将rare class初始预测概率设定为一个prior $\pi$，在本文中$\pi=0.01$。这样设定之后，在class imbalance的情况下，正负两类所占的loss相差就不会太大。这样的initialization会使得训练过程更加smooth。

## RetinaNet

RetinaNet采用FPN的结构，使用ResNet作为backbone。并且延续了RPN的anchor设计。值得一提的是，对于每个level的feature maps，所有的classification subnet和box subnet都是共享参数的。并且class subnet和box subnet都是使用了$3\times3$的卷积核。但是class subnet和box subnet并不共享参数。另外，class subnet的最后一层是sigmoid，而不是softmax。这样处理的目的是为了兼容一个anchor对应多个标签。然后损失函数相当于对每个类别都进行二分类。