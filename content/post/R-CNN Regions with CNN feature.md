---
title: R-CNN

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-07-08"
tags: ["object detection"]
---

# R-CNN: Regions with CNN feature

### 1.人们可以将强大的卷积神经网络应用在自底向上的region proposals上，用于定位和分割对象

### 2.当标记的训练数据稀少时，可以使用监督的一个辅助任务的预训练，加上某一domain-specific的fine tuning，可以获得巨大的表现提升。

---

### 使用RCNN来目标检测

#### module design

分为三个模块：

- region proposals: generate category-independent region proposals.

  使用Selective search生成候选region

- feature vector: a cnn extracts a fixed-length feature vector from each proposed region

  从每个候选区域得到4096维向量。在计算过程中，需要首先将区域中的图像数据转化成与cnn兼容的形式（论文中使用的cnn要求输入size为227 $\times$ 227)。需要时将proposed region warp到需要的size。

- svm: a set of class-specific linear svms

#### test-time detection

在test阶段，每张图片都会生成～2000个region proposals，将每个proposal warp并输入到cnn中，得到固定大小的4096维特征向量。之后使用svm对其打分。给定一张图片的所有的打分的区域，对所有区域按照分值进行排序，然后对每一类都独立使用greedy non-maximum suppression（即如果某一区域和比该区域打分更高的某个区域（相同类别）之间的IoU overlap大于某个threshold，则拒绝该区域）。

#### training

*supervised pre-training* ：使用pre-training，使用auxiliary dataset(ILSVRC2012 classification / 1000类)训练一个分类模型。

*domain-specific fine-tuning* ：将pre-training得到的模型中1000-way的classification layer换成随机初始化的(N+1)-way的classification layer。使用VOC dataset对新的模型fine-tuning。

在训练过程中，首先从原始输入图片中生成～2000个region proposals，接着使用新的模型前向传播，从而fine-tune整个模型。但是，在前向传播之前，还需要确认每个region proposal的label。因为每个proposal的位置和数据集中已标注的对应object的ground-truth box的位置不太可能一模一样，所以在这里需要近似处理一下，得到每个region proposal的label。本文的做法是将和ground-truth box之间IoU大于等于0.5的region proposals的label视为positive，而将剩余的proposals标记为negative（即分为background类）。得到region proposals对应的label之后，需要以此为数据集训练cnn。本文提出一种size为128的mini-batch，在每个batch中，随机挑选32个positive样本和96个negative样本以构建size为128的batch。

一旦特征被提取出来，并且训练的标签也得到了，我们就要对每一类确定linear svm。因为训练数据太大了以致于难以同时装入内存，本文采用了standard hard negative mining的方法。关于此方法，可参考[hard nagetive mining](https://blog.csdn.net/qq_36570733/article/details/83444245)。大致是由于negative样本的数目远远大于positive样本数目，所以为了训练时避免向negative靠近（即防止测试时几乎全都分类到negative），只选取全体negative样本中的一个子集（在本文中取positive样本数:negative样本数为1:3）。但是，如果仅仅只是随机选取的话，可能会导致结果中一些本身是negative的样本被分类成了positive。所以hard negative mining的思路是从全体negative样本中选取hard negative样本作为子集进行训练。其中hard negative样本就是那些很容易被分类成positive的negative样本。