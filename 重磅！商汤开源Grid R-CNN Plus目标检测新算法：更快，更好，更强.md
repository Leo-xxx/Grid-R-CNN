## 重磅！商汤开源Grid R-CNN Plus目标检测新算法：更快，更好，更强

TeddyZhang [CVer](javascript:void(0);) *昨天*

点击上方“**CVer**”，选择加"星标"或“置顶”

重磅干货，第一时间送达![img](https://mmbiz.qpic.cn/mmbiz_jpg/ow6przZuPIENb0m5iawutIf90N2Ub3dcPuP2KXHJvaR1Fv2FnicTuOy3KcHuIEJbd9lUyOibeXqW8tEhoJGL98qOw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> 本文转载自：极市平台

作者简介



**TeddyZhang**：上海大学研究生在读，研究方向是图像分类、目标检测以及人脸检测与识别



前几天，商汤科技开源了Grid R-CNN Plus的代码，相比Faster RCNN，有了很大的提高，今天在介绍Grid R-CNN Plus版本之前，先来回顾下原版的Grid RCNN的动机和创新,然后再看Plus版本是如何改进Grid RCNN的。



![img](https://mmbiz.qpic.cn/mmbiz_jpg/gYUsOT36vfqyfRVeicmLpbicklCMyRNO4NJStvULO1caibuqtowPjTYSR1ib0icicUnHzAaBcXNUx9twEemVGIibK2Ynw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



论文链接：

Grid RCNN: https://arxiv.org/abs/1811.12030

Grid RCNN Plus: 

https://arxiv.org/abs/1906.05688



代码链接：

https://github.com/STVIR/Grid-R-CNN





**回顾Grid R-CNN**

# 算法总览

原版的Grid R-CNN的提出主要侧重于如何让最后的预测框更加的精准，最后的效果也非常的有效，相比于Faster R-CNN+FPN（ResNet50），其在COCO Benchmark的结果在IOU=0.8获得了4.1%AP提升，而在IOU=0.9获得了10.0%的AP提升。



在双阶段算法中，如FPN，RCNN系列、Cascade RCNN、Mask RCNN等算法都有一个共同的特点，就是边框定位模块是相同的。它们将边框定位视作一个回归分支，一般经过几个全连接层对输入的feature map进行预测偏差（proposal或者predefined anchor）。



![img](https://mmbiz.qpic.cn/mmbiz_jpg/gYUsOT36vfqyfRVeicmLpbicklCMyRNO4NOjLqH4MicR69e9fLKNzmKAoTTQqSeachIicrVJG0gcezb7ycF2o5bXRQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



在Grid RCNN中，作者想要用一个全卷积层来代替FCs，以获取更加精确地空间表达。首先，将目标的边框区域分成网格，然后送入FCN中预测相应的网格点的位置，如上图猫咪的9个点。由于全卷积的架构对物体位置敏感是成比例的，所以网格点的位置可以在像素级别来获取。这样，相比之前的回归算法，网络就可以获得更多更精确的空间信息。



但是由于点位置的预测和局部特征没有直接的关系，比如猫咪左上角的点和其相邻的只包含背景的区域拥有类似的特征，这就会增加检测难度。为了解决这个问题，作者设计了一个多点监督的公式，通过在一个网格中定义目标点，可以获得更多信息来减少一些点检测不准确的影响。比如猫咪右上角的点可能不准确，可以通过刚好位于边界的上中点进行监督校准。除此之外，为了充分利用网格点的信息，提出了一种信息融合的策略。具体来说，**对于一个网格点来说，其相邻的特征图都会被收集并融合成一个特征图，这个融合后的特征图用于相应网格点的预测，从而使得这个网格点的位置更加精准。**



总的来说，有以下三点contributions：

1. 提出使用FCN来保留更多空间信息，相比于使用全连接层，也是第一个像素级定位目标的双阶段算法
2. 使用多点监督的方式来避免某些点预测不准确，还提出了网格点相邻特征图信息融合的方式来联合监督点的位置，从而得到更精确的位置。
3. 相比于之前的全连接层的方式进行位置标定，这是一种更好，更严格的网格监督的定位方式，特别对于现在COCO竞赛中要求的IOU=0.75以上。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



# 与CornerNet的比较

CornerNet是一个用成对的关键点来确定目标边框位置的单阶段算法，他是一个bottom-up的设计方式，也就是说首先检测出来所有的边框关键点，然后进行分组从而对目标的边框进行定位。而Grid RCNN和它是不一样的思路，其是一个top-town的two stage的方式，首先在第一阶段确定了目标的大致区域，即regionproposals。接着，Grid RCNN主要的关注点是如何更加精确的定位目标的关键点，从而提出了grid points locations。



# GridGuided Localization

在图中为一个3x3的网格点的形式，包括四个角点、四个边的中点以及中心点。每个region proposal由ROI Align生成，固定尺寸为14x14，接着经过8个带dilation操作的空洞卷积用来增加感受野，然后再使用2个反卷积操作使特征图尺寸为56x56。因此每个检测分支都会输出一个尺寸为56x56的heatmap，逐像素的进行Sigmoid函数操作后获得相应的概率图，并且每个heatmap都有一个对应的监督图，也就是groundtruth map，这个真实图的每个grid point的真实位置用一个5像素的十字进行标记。



使用如下公式将heatmap中Gridpoint的位置（![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)）映射到regionproposal( ![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==))：





![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



其中p和o分别代表region proposal和heatmap。



然后我们假设一个边框的四个坐标为，这个就感觉有点像CornerNet的坐标定义了，怪不得要专门对比一下这两个网络的区别。但是由于分成了9个grid points，可能在进行预测时，原本处于同一水平线的点不在同一水平线，因此取平均值消除误差影响。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



假设我们要图上猫咪的预测边框B中的，那么猫咪上方有三个点，对其进行概率加成求和后取平均就可以得到我们的回归对象.



**Grid Points Feature Fusion**

由于网格点具有内部相关性，因此可以相互之间进行校准，从而减少整体偏移，因此作者设计一个空间信息融合模块来代替原来直接求平均的方法，后者导致很多特征图信息没有使用。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



上图为网格点特征融合的结构，图b为GridRCNN的关键点信息传播方向，为了可以分辨不同点的feature map，我们使用NxN组卷积和对最后一个特征图提取特征，因此每个特征图都很明确对应一个gridpoint。对应每一个grid point，与这个点距离1个单位网格长度的点叫做源点，并定义第i个grid point为 ![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)，因此第j个源点经过三个连续的5x5卷积后，然后与原来点的特征图进行融合，这就是图a的情况（一阶融合策略）。当然图b是允许与该点距离为2个单位长度的点进行融合（二阶融合策略）。其中三个5x5的卷积就相当于信息转换算子![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



# ExtendedRegion Mapping

由于全卷积层的特性，最后输出的heatmap的空间信息很自然的与原始图像所产生的region proposal相关，但是一个region proposal并不一定包含整个目标，也就是说一些真实框的grid points可能在region proposal外部，因此不能再监督图上进行标注。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



如上图，小的白色的可能是通过RPN+ROI Align产生的region proposal，很明显，没有包含其grid points，这样就会使训练过程效率非常低。



因此，一个自然的想法就是扩大这个proposal区域，这个区域可以包含大多数的grid points，但是这样也会引入一些无效的背景区域或者其他的目标，经过试验测试，这种策略没有收益却损害了小目标的精度。



为了解决这个问题，我们修改了输入heatmaps和原始图区域的对应关系，如下公式所示，这样就可以得到上图最大的那个dashed box。注意，只有IOU大于0.5的region proposal才可以进入到Grid branch。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



# 实验结果

## **对比实验**

**1.  多点监督策略**

对比了传统的回归算法，在2-point使用真实框的左上角和右下角的点，在4-point中增加了另外两个角点的监督，9-point就是论文示例中的普通3x3grid。其最后的结果显示，越多的grid point监督效果越好。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



**2.   网格点特征融合策略**

对比的baseline为没有进行网格点融合的形式，first order是进行了一阶融合，second order是在一阶融合的基础上增加了二阶融合的措施。特别的，可以看到这种策略对于AP.75的影响要大于AP.5，这说明特征融合策略对于边框定位准确度十分有效。





![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



**3.  增大区域映射**

对比了没有做任何措施、直接扩大region proposal以及使用该表映射关系的方法，可以看到直接扩大region proposal虽然对大物体检测区域有提高，但由于引入了背景和其他目标，对小目标的检测却十分不利。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



**4.  对比COCO不同IOU阈值的结果**

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



从上图可以很容易看出，Grid RCNN不改变网络结构，只是通过改变最后回归分支的结果和方法，从而达到了很精确的边框定位。虽然在IOU 0.5阈值下略有下降，但这个思路值得学习，特别追求IOU阈值比较精准的任务中。





**提升Grid R-CNN Plus**



虽然原版的Grid R-CNN对FasterRCNN做了很多精度上的优化，但是速度却慢于Faster RCNN，于是作者就其进行改进并进行了开源，开源地址见开头！Res50+FPN作为主干网络的Grid R-CNN Plus达到了40.4%的AP，相比FasterRCNN，精度增长了3.0%，速度却几乎相同。



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



# 算法总览

## **GridPoint Specific Representation Region**

对于Grid RCNN Plus来说，最大的提升就是网格点的特征表达区域，由于只有正样本（IOU>0.5）才会被送入Grid branch，因此有些真实标签会被限制在监督图的一个小区域内。如下图所示：



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



在一个3x3的grid point中，真实标签只会出现在监督热图的左上方区域，但这样是不对的，对于一个物体来说，它的所有的grid points共享一个相同的特征表达区域。为了解决这个特征表达区域的问题，首先，将grid branch的输入尺度从原来的56x56降低为28x28，对于每个grid point，新的输出代表了原来大概四分之一的区域。经过这样处理后，每个grid point的表达可以近似的视为一个归一化的过程。



# LightGrid Head

由于最后的输出尺度降低了一半，那我们可以同时将grid branch中的其他特征图分辨率也降低，比如14x14到7x7。细节来说，通过前面的RPN+ROIAlign产生一个固定的feature map 14x14，接着使用一个步长为2的3x3卷积核，然后再使用7个步长为1的3x3卷积核从而产生7x7分辨率的特征图。紧接着我们将这个特征分成N组（默认为9）,每一组关联一个grid point，接着使用两个组反卷积将特征图尺度变为28x28，注意group deconvolution可以加速上采样的过程。



另外一个好处是，由于我们对每个grid point的表达进行了归一化，因此他们变得更加closer,导致在特征融合时不需要使用很多的卷积层来覆盖这个间隙。在Plus版本，只使用了一个5x5 depth-wise卷积层来代替原来的3个连续的卷积层。



# Image-acrossSample Strategy

由于grid branch在训练时只使用正样本，所以不同采样batch正样本数量的不同会对于精度产生影响，比如，有些图像的正样本很多，但有些图像的正样本数很少。在Plus版本，作者使用了跨图片的采样策略，具体讲，从两个图片中一共采集192个positive proposal，而不再是每张图片采集96个positive proposal。这样就会使训练更具有鲁棒性以及精度提高。



**NMS Only Once**

原来的Grid RCNN需要两次NMS，第一次是proposal的生成，只选择Top 125个样本进行边框矫正，第二次是做最后的classification，作者发现，尽管只是一小部分的proposal，进行80类的NMS还是很慢，所以在Plus版本，直接移除了第二个NMS，同时将第一个NMS的IOU阈值设置为0.3，分类阈值设置为0.03，只选择100个proposal进行进一步的分类和回归。

#  

# 实验结果



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



可以看到，Plus版本更快，更好，这些策略还是值得学习，研究的！





**CVer-目标检测交流群**



扫码添加CVer助手，可申请加入**CVer-目标检测群。****一定要备注：****研究方向+地点+学校/公司+昵称**（如目标检测+上海+上交+卡卡）

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

▲长按加群



这么硬的**论文分享**，麻烦给我一个在**在看**



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

▲长按关注我们

**麻烦给我一个在看****！**

[文章转载自公众号![极市平台](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7yhttE6xvg7Z4f6OyQibAB7ehiawBQP7OcMrpsEy1YwRng/0) **极市平台** ](https://mp.weixin.qq.com/s?__biz=MzUxNjcxMjQxNg==&mid=2247489955&idx=2&sn=519e828434b91bdc4c5c9078c8c54b55&chksm=f9a26b2cced5e23acfdfa685c6157101bbeb99ebe8c5ab1fad0bffeaa29d6519de16c4cf90cb&mpshare=1&scene=1&srcid=&key=90581f21d61583cc21ae31128859a815a55baaf6863efa0a9e36ae70092a6acc4a907ea98c4f6faa7932b5fd9e316d89983f97e9310729bd128d07b2f6ac9d25b86737fd798404e5f3b748fcf69fbc61&ascene=1&uin=MjMzNDA2ODYyNQ%3D%3D&devicetype=Windows+10&version=62060833&lang=zh_CN&pass_ticket=gTcGMiRI%2FrgdalBpEedBJUR9Xe7se%2B0GfGmnO0zGcIM9Lj2iEqzyYiPvMtdKmsPA##)





![img](https://mp.weixin.qq.com/mp/qrcode?scene=10000004&size=102&__biz=MzUxNjcxMjQxNg==&mid=2247489955&idx=2&sn=519e828434b91bdc4c5c9078c8c54b55&send_time=)

微信扫一扫
关注该公众号