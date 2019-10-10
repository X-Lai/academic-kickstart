---
title: YOLO v2

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-09-11"
tags: ["object detection"]
---

# YOLO v2: YOLO9000

和YOLO v1相比，YOLO v2做出了一些improvements，使得目标检测更快而且效果更好。

## Better

1.Batch Normalization：获得more than 2% improvement；

2.High Resolution Classifier：YOLO用的classifier network的输入原来是224*224的，只是后来为了检测而提升了448\*448，这就意味着YOLO在训练的时候还需要adjust到新的输入分辨率上，造成训练的时候效果不好。而对于YOLO v2，在一开始的时候它就先在ImageNet上对classification network用448\*448的分辨率进行fine tune 10个epochs 。这样就使得这个网络有时间来适应新的分辨率。然后再对得到的network用detection来正式地fine tune。这样可以获得接近4% mAP的提升。

3.采用anchor boxes。像Faster R-CNN一样使用anchor，这样YOLO v2只需要预测offset，而不是直接预测一个box coordinates。本文提到比起预测coordinates，预测offsets更简化了问题，也使得网络更易于训练。同时，为了得到higher resolution output，减少了一层pooling layer。并且在预测anchor的时候，与YOLO不一样的是，YOLO v2会为同一个grid cell的不同的box predictor都预测对应的class，这样就使得输出的channels数量是$H\*W\*num\\_anchors*(4+1+num\\_classes)$。使用anchor虽然mAP有一点点下降，但是recall上升了，所以这样做提供了更大的优化空间。

4.Dimension Clusters：在Faster R-CNN里，anchor的size和aspect ratio都是hand-picked，所以可能会误差较大。本文提出对训练集的所有的ground-truth box进行k-means聚类，来挑选k类不同size或aspect ratio的anchor。表示一个box和一群boxes的距离使用IOU如下表示：$d(box, centroid)=1-IOU(box, centroid)$，这样就使得被聚在一类的anchor boxes之间的IOU很大，甚至基本重合。所以，聚类产生的结果可以得到k种能够代表几乎所有boxes的size或aspect ratio了。通过比较训练集的bounding boxes与所用anchor之间average IOU，发现使用cluster产生的anchor时，k=5的效果和使用hand-picked anchors时k=9的效果是差不多的。由此可以发现dimension cluster的优势。

5.Direct Location Prediction：使用了anchor之后，如果按照faster R-CNN的方法，在预测box的中心点(x,y)的时候，是用$x=t^x\*w_a+x_a,y=t^y\*h_a+y_a$计算得来的。这样的话，会造成在训练的开始阶段，预测的box在image的每一处都有可能出现，这种unconstrained的方法会导致模型的instability。本文就提出预测相对于grid cell的location的offsets。这样的话就可以将输出值限制在(0,1)之间。具体公式如下：$x=c_w\sigma{(t_x)}+c_x,y=c_h\sigma{(t_y)}+c_y,w=p_we^{t_w},h=p_he^{t_h}$，其中$c_x,c_y,c_w,c_h$分别表示grid cell的左上角的横纵坐标以及宽和高，$p_w,p_h$分别表示对应anchor的宽和高。$\sigma{()}$表示sigmoid函数。实验表明，这样做可以获得接近%5的提升。

6.Fine-Grained Features：为了得到不同resolution的feature map，从而使得提取的特征更加完善，YOLO v2将前一个输出为26*26的layer作为passthrough layer，然后在具体实现时将它切成多个13\*13的feature maps，这样再与之前的13\*13的输出堆叠成多个feature maps一起进行预测。这样可以获得大约1%的提升。

7.Multi-scale Training：这样可以使得network在不同resolution下预测detection，使网络更加generalized。

## Faster

提出Darknet-19作为backbone，使得计算量更小。

## Training

如何标记ground-truth: 在YOLO v2的结构里，输出有$H\*W\*num\\_anchors*(4+1+num\\_classes)$个channels，因为和YOLO v1不一样，它对每个region都预测其class。那么输出一共有3类，分别是：

1.localization：每个localization输出一个四元组$(\sigma{(t^x)}, \sigma{(t^y)}, t^w, t^h)$，其中$\sigma{(t^x)}=\frac{x-x_c}{w_c}, \sigma{(t^y)}=\frac{y-y_c}{h_c}$，$x,y$分别表示预测的bounding box的中心点的横纵坐标，$x_c, y_c$分别表示对应grid cell的left top的横纵坐标，$w_c, h_c$分别表示grid cell的宽和高; $t^w=ln{\frac{w}{w_a}}, t^h=ln{\frac{h}{h_a}}$，$w,h$分别表示预测的bounding box的宽和高，$w_a, h_a$分别表示对应的anchor的宽和高。所以通过以上公式即可在预测的四元组$(\sigma{(t^x)}, \sigma{(t^y)}, t^w, t^h)$和bounding box的四元组$(x,y,w,h)$之间相互转换。所以在localization方面，直接使用给定的ground-truth boxes来得到另一种表达形式；

2.objectness：对于每一个localization，YOLO v2都会预测一个对应的objectness $\sigma{(t_o)}$。而且YOLO v2设定每个ground-truth box只由一个anchor来负责。首先，对于每个ground-truth box，先确定预测它由它的中心点落在的那个grid cell负责；其次，找到与该grid cell对应的所有anchors中与该ground-truth box的IOU最大的那一个anchor（待定，不确定是由anchor本身来算IOU，还是anchor对应的那个prediction box来算IOU），即由该anchor来负责预测该ground-truth bounding box。另外，对于那些与每个ground-truth box的IOU都小于threshold的prediction boxes，它们对应的ground-truth objectness设置为0。

3.classification：对应于每个localization，YOLO v2还会预测它被分到哪一类。只需要考虑那些ground-truth boxes对应的localization即可，因为只有这些位置上的classification才会在计算loss的时候用上。

