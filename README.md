# SCMN_README
## 项目简介
本算法实现了高性能的半监督视频对象分割，主要使用了添加了Spatial Constraint模块、Segmentation Head的时空记忆网络。时空记忆网络即通过利用存储过去带有mask的帧的外部存储器来学习从所有可用资源中阅读相关信息，而查询中的当前帧则使用存储器中的掩码信息进行精细分割。
## 解决方案
在本算法中，使用第一帧中给出的ground truth从第二帧开始顺序处理视频帧。在视频处理期间，我们将带有mask的过去帧（在第一帧给出或在其它帧处估计）视为存储帧，将没有mask的当前帧视为查询帧。通过Memory Encoder和Query Encoder分别获取存储帧和查询帧的key-value对，通过在Space-time Memory Read模块中将查询帧和存储帧的关键特征图上的每个像素在视频时空上紧密匹配，然后根据相对匹配分数来寻址存储帧的value特征图，组合作为输出。在其后，我们利用ASPP模块和优化模块来提高性能。接着，我们添加了Spatial Constraint模块，它利用编码器和ASPP模块生成的previous predict和current embedding来学习Spatial Prior，指导模型过滤错误。最后，我们使用Decoder获取输出，并为查询帧重建mask。
## 具体算法
### Memory Stage
#### Memory Encoder
通过Memory Encoder将存储帧编码为key-value映射对，我们将ResNeSt-50作为Memory Encoder和Query Encoder的backbone network。Memory Encoder同时采用RGB图像和object mask，输出key-value映射。key用于寻址，即计算查询帧与存储帧之间的相似性以确定何时何地从中检索相关value；value存储用于产生mask estimation的详细信息（例如目标物体和物体边界），即存储详细的外观信息以供解码准确的object mask。如果有多个存储帧，则每个存储帧都独立地嵌入到key-value映射中，然后将来自不同存储帧的key-value映射沿时间维度堆叠，Memory Embedding的输出是一对3D的key-value映射。
### Predict Stage
#### Query Encoder
通过Query Encoder将查询帧编码为key-value映射对。Query Encoder仅采用图像作为输入，其结构与Memory Encoder相同，输出是一对2D的key-value映射。
#### Space-time Memory Read
在该模块中，首先通过测量查询帧和存储帧的key的所有像素之间的相似度来计算权重，再通过将存储帧中每个时空位置的key与查询帧中每个空间位置的key进行比较，以non-local方法执行相似性匹配，然后通过加权求和来检索存储帧的value，并将其与查询帧的key连接在一起。
#### Segmentation Head
Spatial Constraint模块旨在正确地捕获目标对象，但这还不足以获得高质量的结果。由于在多目标的情况下，目标对象的大小各异且通常会发生变化，因此我们提出了Segmentation Head来解决此问题并提高分割质量。具体来说，我们在进行Space-time Memory Read操作后应用了ASPP模块，使用了三个扩张率分别设置为2、4和8的并行空洞卷积层后进行上采样操作，再连接采用自Basnet模型中的带有skip connection的模块，进行解码操作。我们应用soft aggregation来合并多目标预测，为了进一步提高性能，尤其是边界处的分割，我们采用了基于编码器-解码器结构的优化模块。将软聚合之前的特征图作为优化模块的输入，然后使用3x3卷积层和ReLU函数对其进行下采样，再将其上采样到原始分辨率，并再次通过soft aggregation进行合并。
#### Spatial Constraint Module
我们引入Spatial Constraint模块以实现相邻帧之间的空间一致性，消除外观混乱和由相同类别的相似实例引起的错误预测。具体做法是，将前一帧的预测mask与当前帧的embedding串联在一起，接着采用具有3×3核尺寸和S形函数的卷积层来生成Spatial Prior特征图，再用它乘以当前帧的embedding。
#### Decoder
Decoder模块用于获取前序步骤所得的输出，并重建当前帧的object mask。主要使用了多个优化模块将压缩的特征图每次按比例放大2倍，每个阶段的优化模块都通过skip connection获取上一阶段的输出以及来自Query Encoder的相应比例的特征图。最后通过卷积层和softmax运算来重建object mask。
