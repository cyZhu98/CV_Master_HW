# TODO: 补充剩余实验的结果
### 基础任务

* 基于Faster-RCNN 在我们提供的训练集上训练模型并优化性能, 观察训练过程损失变化
* 在上述实验的基础上, 复现 FSDet [论文](https://arxiv.org/abs/2003.06957) [代码](https://github.com/ucbdrive/few-shot-object-detection) 并测试性能, 与(1) 对比优化过程和尽可能提升性能

### 提升任务

* 能否通过所学内容优化上述实验中的RPN模块, 给出前后测试集指标对比(可截图)和优化思路
* 能否使用 Prototypical Network 思路优化小样本检测的分类分支
* 能否使用 One-Stage 方法来实现性能较好的小样本检测器

# 基础任务1：

刚训练的时候下载了100mb的resnet50的预训练权重

训练日志保存在[train.log](/hw2/pic/train_log.txt)中，后面更改了学习率截断的epoch


总体的loss肯定是不断下降的，然而回归loss，主要是rcnn的bbox_loss不降反增

观察源码，训练前，ResNet50的第一层卷积和第一个Block(通道为64的3个Bottleneck)被冻结

Faster R-CNN预训练模型最后的回归和预测层的参数没有读入(输出的类别不一样)。

Batch_size设为1，单卡训练，lr为0.02/16，可能是初始化结果比较好，rcnn_bbox一开始比较低。回归Loss都是L1 Loss
![image](/hw2/pic/test.png)

最终[测试结果](/hw2/pic/test_log.txt)（第36个epoch）*AP50为0.128*

将 models/faster_rcnn.py中的self.rpn_smooth_l1_beta和self.rcnn_smooth_l1_beta设为1，回归loss更改为smooth l1,但出了bug，beta改为1会报错
![image](/hw2/pic/bug.png)

其他参数 参考PaddleDet和MMDet应该都是最优参数了

其他尝试：backbone换成ResNet101，效果不佳

# FSOD (/code/train_fsod.py):

文章思路：用大量的Base Images训练出一个模型，再用少量的Base Shots和Novel Shots微调预训练模型中的最后一层:box classifier 和box regressor。

作者还提出了一个基于cosine similarity的分类器，在微调阶段替代线性层。

对于COCO数据集，作者用了60类作为base class，20类作为novel class。在本次作业中，用的是object365数据集，理应先在object365数据集上用Base Class训练出一个模型再用所给的训练集，加上少量的novel class进行微调。
由于作业提供的数据就这些，所以我偷了懒，直接用COCO预训练模型，用所给训练集做fine-tune，这样做的缺点是最后一层输出层的分类数不一样，所以只能固定前面的权重，最后一层重新训练。

[测试结果](/hw2/pic/fsod_test.txt)AP50 = 0.204, AP75 = 0.109, mAP0.5:0.95 = 0.112，比上边的Faster RCNN要高，但是由于做法是有问题的，所以离paper的结果要低不少。
![image](/hw2/pic/fsod_test.png)
)

在课程中老师指出这种基于Fine Tune的做法是有问题的，主要在于RPN的权重被冻结，会把Novel Class当成是背景处理，我对此最简单的改法是不冻结RPN模块的参数/TODO

# 提升任务
architecture更换为RetinaNet,backbone = ResNet50，参数不做更改 [训练日志](/hw2/pic/retina_train.txt)
[测试结果](/hw2/pic/retina_test.txt)--这次log还没删tqdm，找结果比较麻烦。最佳mAP在epoch28，之后开始过拟合：AP50 = 0.105，mAP0.5:0.95 = 0.073,AP75 = 0.081，还是要低于Faster RCNN的
