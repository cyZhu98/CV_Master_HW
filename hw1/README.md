# 作业说明
## 1 作业内容 
作业划分为不同的难度档次，对应不同的星级(满分为5星)，可根据自己的时间和能力自主选择任务(基础任务星级不可叠加，按完成的最高星级计算成绩；附加任务星级可叠加：    
- **基础任务**   
    【2 ⭐】: 实现Freeanchor/FCOS任意一种    
    【3 ⭐】: 实现ATSS    
    【4 ⭐】: 实现Auto-assign/PAA(简化版)任意一种    
- **附加任务**  
    【0.5 ⭐】：优化基础任务中的label assginment，给出前后测试集指标对比(可截图)和优化思路、自己的理解  
    【0.5 ⭐】：实现至少两种不同星级的基础任务并比较他们的label assignment(可利用统计、可视化等手段)，给出自己的理解  

## 2 作业要求 
数据集：cocomini（平台上已存放cocomini数据集，同时我们也提供了数据集<a href="https://1drv.ms/u/s!Amprt__M3WbSgxjhpLW1HLotXGTG?e=Hr3Y0B" target="_blank">下载地址</a>满足离线训练的需要）    
深度学习框架：megengine  
Codebase：Retinanet（站在巨人肩膀上，复用去年 cv master repo， 内容详见<a href="https://studio.brainpp.com/project/2921" target="_blank"> 往届课程作业</a> 和 <a href="https://www.bilibili.com/video/BV1Xp4y1r7WV?p=2" target="_blank"> 往届课程回顾</a> ）  
其他：**本作业旨在辅助同学理解目标检测中的label assignment，无需修改backbone/训练方式等无关部分**   


## 3 作业提交
作业截止时间：**2021-05-21 22:00:00**，请务必在截止日期之前完成提交动作  
作业打开方式：**请先fork本项目，然后在生成的项目中进行实验，实验完成后可以选择版本进行提交**  
其他：**提供github上自己管理的repo**，让我们更清楚了解你作出的努力；在作业中适当添加自己的思路、理解

# 作业完成
## 基础任务 基于MegEngine/Models
* 实现FCOS
* 实现ATSS

## 附加任务

FCOS的架构如下图，代码为code/models/FCOS.py
![image](/hw1/pic/fcos_backbone.png)

label assignmetn的策略是只要在bounding box中的点都是正样本的候选，之后通过centerness来进一步确定positive sample

和给出的RetinaNet代码不同点：所有的anchor都换成point。
首先根据不同level的feature map 生成point
```python
self.anchor_generator = layers.AnchorPointGenerator()  # 位于layers/anchor.py
```
定义point的decoder和incoder操作，在FCOS中是gt bos的TBLR变换
```python
self.point_coder = layers.PointCoder()  # 在box_utils中定义，定义了encode和decode
``` 
backbone和FPN与retinaNet相同
最后定义图片中后半部分，即FCOS Head

```python
self.head = layers.PointHead(cfg, feature_shapes)  # 用于分类和回归，外加一个centerness打分，在layers/point_head.py中定义
```

**point_head.py**中，两个通道各4个卷积block

```python
self.cls_subnet = M.Sequential(*cls_subnet)
self.bbox_subnet = M.Sequential(*bbox_subnet)
```

再定义3个输出，分别是分类Score，bounding box回归 ，作者新定义的center-ness，作用是纠正point回到gt box的中心

```python
self.cls_score = M.Conv2d(in_channels, num_anchors * num_classes, kernel_size=3, stride=1, padding=1)
self.bbox_pred = M.Conv2d(in_channels, num_anchors * 4, kernel_size=3, stride=1, padding=1)
self.ctrness = M.Conv2d(in_channels, num_anchors * 1, kernel_size=3, stride=1, padding=1)
```

**这里ctrness分支论文中说是和分类分支平行的**，看代码里变成和回归分支一起了，**在paper后面的实验部分说明这样做的精度更高**

```python
def forward(self, features: List[Tensor]):
		logits, offsets, ctrness = [], [], []
    for feature, scale, stride in zip(features, self.scale_list, self.stride_list):
        logits.append(self.cls_score(self.cls_subnet(feature)))  # 分类score
        bbox_subnet = self.bbox_subnet(feature)
        offsets.append(F.relu(self.bbox_pred(bbox_subnet) * scale) * stride)  
        ctrness.append(self.ctrness(bbox_subnet))  # 看论文这应该是在cls_subnet后面
    return logits, offsets, ctrness
```

ctrness的定义：

![](https://latex.codecogs.com/gif.latex?centerness=\sqrt{\frac{min(l^*,r^*)}{max(l^*,r^*)}\times&space;\frac{min(t^*,b^*)}{max(t^*,b^*)}})

```python
ctrness = F.sqrt(
  F.maximum(F.min(left_right, axis=1) / F.max(left_right, axis=1), 0)
  * F.maximum(F.min(top_bottom, axis=1) / F.max(top_bottom, axis=1), 0)
)
```
[测试log](/hw1/pic/fcos_test.txt)AP=0.236,AP50=0.394,AP75=0.245

总结一下paper中提到的涨分技巧：1.ctrness分支放到回归那边 2.用GIOU Loss 3.只采样gt box中心部分的点作为正样本 4.标准化回归值(t,l,b,r)

**其中，用GIOU Loss很容易理解，而技巧3的思想和ATSS也比较相似，都是为了获得离gt box中心更近的样本**

ATSS:

**code/models/ATSS.py**

ATSS中提出的策略：对于所有gt box，对于所有的预选框，根据L2距离，选取离中心最近的k个anchor。

计算这些anchor的IOU均值和标准差，两者之和作为IOU阈值，符合阈值的并且中心在框内的anchor作为正样本存入，其余的anchor作为负样本。官方实现是基于FCOS的，结果确实是比FCOS稍微高一点，优化的话除了FCOS的参数，只有一个超参k

代码没来得及看

[测试log](/hw1/pic/atss_test.txt)AP=0.240, AP50=0.390,AP75=0.251	



