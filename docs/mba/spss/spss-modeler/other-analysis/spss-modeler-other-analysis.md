# 13：神经网络网络

神经网络是一种试图模拟生物神经网络的结构和功能的数学模型或计算模型。类似于人脑中神经元的结构中的多个神经元通过轴突链接，神经网络也是由一组相互链接的节点（神经元）和有向链构成的。人的大脑在同一个脉冲的反复刺激下，通过改变神经元之间的神经连接强度来进行学习，神经网络也是基于类似机制。通过反复训练集进行学习，改变结点间的链接权重，从而识别出数据中发的复杂模式

## 神经网络介绍

[神经网络介绍]: http://luluwanlong.cn/archives/spss-modeler-introduction-to-neural-networks

## 类神经网络分析参数设置说明

![image-20230126071946496](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126071946496.png)

- 字段

- 构建选项

  ![image-20230126072053167](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126072053167.png)

  - 目标：与C&R内容一致 参考

    [决策树分析]: http://luluwanlong.cn/archives/spss-modeler-analysis#12.3.4-%E5%8F%82%E6%95%B0%E8%AE%BE%E7%BD%AE%E8%AF%B4%E6%98%8

  - 基本

    ![image-20230126072407165](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126072407165.png)

    - 神经网络模型
      - 多层感知器：
      - 径向基函数：
    - 隐藏层
      - 自动计算单元格格数：只构建单个隐藏层神经网络，由SPSS Modeler 自动确定最佳的隐藏层个数
      - 定制单元格数：手动指定隐藏层的结构，在SPSS Modeler 中，最少必须包含一个隐藏层，最多包含2个隐藏层
        - 隐藏层1：
        - 隐藏层2：

  - 中止规则

    ![image-20230126072800364](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126072800364.png)

    - 使用最大训练时间
    - 使用最大训练周期数量
    - 使用最低准确率

  - 整体

    ![image-20230126072912068](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126072912068.png)

    - 
    - 主要是集成学习算法的相关设置

  - 高级

    ![image-20230126073018956](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126073018956.png)

    - 过度拟合防止集合：从训练集合中抽取独立的样本作为验证集，用于错误率的检验
    - 复制结果：由于验证集的抽取是使用随机抽样方式，因此通过设置随机种子可以保证重现结果
    - 预测变量中的缺失值：
      - 成列删除：若某个记录样本在输入变量中存在缺失值，则在建模阶段将排除该样本
      - 插补缺失值：若某个记录样本在输入变量中存在缺失值，则在建模阶段将对该样本的缺失值进行插补。对于连续型变量，将插补最小值和最大值的平均值；对于分类型变量，将插补最常出现的类别

- 模型选项

- 注释

![image-20230126074322239](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126074322239.png)

![image-20230126074335722](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126074335722.png)

![image-20230126074351862](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126074351862.png)

![image-20230126074522888](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126074522888.png)

![image-20230126074613557](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126074613557.png)



![image-20230126074845851](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230126074845851.png)

# 14：聚类分析

##  聚类方法概述

[聚类分析介绍]: http://luluwanlong.cn/archives/spss-modeler-introduction-to-cluster-analysis

## K-Means分析参数设置说明

![image-20230128225744333](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230128225744333.png)

- 字段

- 模型

  - 模型名称
    - 自动
    - 定制
  - 使用分区数据：如果使用了分区节点后预定义了分区字段，将使用训练集进行模型训练
  - 聚类数：聚类数目k
  - 生成距离字段：输出模型将输出距离字段，该字段计算量每个样本距离所对于聚类中心的距离
  - 聚类标签：生成聚类群的命名方式，包含有字符串及数字格式
  - 优化：根据具体建模需求，选择是否需要提高模型性能

- 专家

  ![image-20230128230028271](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230128230028271.png)

  - 模式
  - 停止
  - 集合编码值

- 注解

  ![image-20230128230647030](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230128230647030.png)

​	![image-20230128230719403](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230128230719403.png)![image-20230129082504241](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129082504241.png)





# 15：KNN分类器

## KNN 学习方法介绍

## KNN分析参数设置说明

![image-20230129084049128](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129084049128.png)

- 目标

  - 您要执行那种类型的分析
    - 预测目标字段：执行分类预测任务，根据knn算法找到预测目标值
    - 只识别最相邻的值：不进行预测，只找到每个样本的最近邻
  - 您的目标是什么
    - 平衡速度和精确度：算法将在一个较小范围（3-5）选择不同的k值进行校验分析，以效果最好的k值作为最终模型的参数
    - 速度：knn算法的计算量较大，为了优化速度，将选择固定的k值（默认为3）进行校验分析
    - 准确性：算法将在一个较大的范围（3-7）选择不同的k值进行校验分析，以效果最好的k值作为最终模型的参数
    - 自定义分析：自定义算法设置

- 字段

- 设置

  - 模型

    ![image-20230129084724600](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129084724600.png)

    - 模型名称
      - 自动
      - 定制
    - 使用分区数据
    - 为每个分割构建模型
    - 标准化范围输入：由于存在距离维持，与k-means算法类似，为了解决由于变量数量级带来的不良影响，建议在分析计算前对数据进行标准化处理
    - 使用观测值标签：可以在输出模型中“模型查看器”结果中为每个样本添加识别标签，无此选项，将以每个样本的行号作为标签
    - 识别焦点记录：可以在输出模型中“模型查看器”结果中自动选择该字段的点作为焦点进行查询

  - 相邻元素

    ![image-20230129085206674](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129085206674.png)

    - 最近邻元素个数（k）
      - 指定固定k:指定固定的k值作为最近邻元素个数
      - 自动选择k：指定k的范围，knn算法将在此范围内选择最合适的k值作为最终算法的参数，如果在特征选择中，选择了特征选择方法，那么将对范围中的k值计算其误差，选择获得最低误差的k值作为最终参数；如果没有选择特征选择方法，那么将使用交叉校验的方式获得最佳的k值
    - 距离记录
      - Euclidean测量：选择欧式距离作为样本之间的距离计算公式（累计平方和）
      - 城市街区策略：选择曼哈顿距离作为样本之间的距离计算公式
    - 计算距离时按照重要性加权特征：在计算样本间距离时将以变量的重要性作为权重进入公式进行计算。重要性越大的变量将拥有更大的权重
    - 范围目标的预测
      - 最近邻元素值平均值：当预测目标是连续型变量时将使用最近邻样本的平均值作为最终预测目标
      - 最近邻原始值中值：当预测模板是连续型变量时将使用最近邻样本的中值作为最终预测目标

  - 特征选择

    ![image-20230129090312839](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129090312839.png)

    - 执行特征选择：knn算法将采取类似回归分析中前进法的方法进行模型特征选择。在变量加入模型前，算法将所有未加入模型的变量在加入模型后的误差下降值，并从中选择使用模型误差下降最大的变量加入模型，如此往复直到满足中止条件为止
    - 强制条目：手动指定某些变量强制进入模型
    - 终止条件
      - 在选择了指定数目的特征时中止：除了被强制纳入模型的变量，向前选择也将加入其他变量用于模型误差的改善。此选择，knn算法将加入固定数量的变量进行模型之中
      - 在绝对误差比率变小或等于最小值时中止：指定最小变化阈值，当模型的误差在加入新的变量后的变化小于或等于此阈值时，将中止算法输出结果，较小的阈值将生成拥有更多变量的模型

  - 交叉验证

    ![image-20230129091103870](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091103870.png)

  - 分析

    ![image-20230129091122030](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091122030.png)

- 注解

![image-20230129091345753](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091345753.png)

![image-20230129091425845](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091425845.png)

![image-20230129091435157](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091435157.png)

![image-20230129091449209](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091449209.png)

![image-20230129091500876](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091500876.png)

![image-20230129091521733](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129091521733.png)

# 16：关联分析

## 关联分析介绍

## Apriori 分析参数设置说明

![image-20230129092553998](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129092553998.png)

- 字段

- 模型

  - 模型名称
    - 自动
    - 定制
  - 使用分区数据
  - 最低条件支持度：指定关联分析中前项的最小支持度阈值，默认10%
  - 最小规则置信度：指定关联分析中规则的最小置信度阈值，默认80%
  - 最大前项数：指定前项的最大项目数量，通过约束最大前项数量，可以避免生成过于复杂的规则以减少运行时间
  - 仅包含标志变量的true值
  - 优化
    - 速度
    - 内存

- 专家

  ![image-20230129093055003](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129093055003.png)

  - 模式
    - 简单
    - 专家
  - 评估测量
    - 规则置信度
    - 置信度差
    - 置信度比率
    - 信息差
    - 标准化卡方
  - 容许没有前项的规则

- 注解

  ![image-20230129093600297](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129093600297.png)

  ![image-20230129093701614](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129093701614.png)

  

# 17：自动建模

## 自动分类

在“自动分类”节点中，会将SPSS Modeler 里面所有的分类算法都包含在其中，运行的时候，会按照设置

![image-20230129100449937](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129100449937.png)

### 分析步骤

#### 数据源节点

​	![image-20230129100635199](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129100635199.png)

#### 类型节点

![image-20230129100652182](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129100652182.png)

#### 分区节点

![image-20230129100701483](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129100701483.png)

#### 自动分类节点

![image-20230129100722334](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129100722334.png)

- 字段

- 模型

  ![image-20230129100814550](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129100814550.png)

  - 模型名称
    - 自动
    - 定制
  - 使用分区数据：按照分区的设置随机选择训练数据及测试数据
  - 为每个分割构建模型：按照拆分的字段分别构建模型
  - 模型排序依据：针对所有参与计算的模型，按什么标准排序得到前3名
    - 总体精确度
    - 曲线下面积
    - 利润
    - 增益
    - 字段数
  - 模型排序方式：按照是训练分区还是测试分区的模型准确性
  - 要使用的模型数量：默认自动选择3个最优模型
  - 计算变量重要性：输出模型中变量的重要性，并排序
  - 利润条件
    - 成本
    - 收入
    - 宽度（权重）
  - 增益条件
    - 用于增益计算的百分位数

- 专家

  ![image-20230129141957185](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141957185.png)

  - 选择模型
    - 所有模型：默认情况下，会选择所有算法运行
    - 在Modeler上运行（逐个指定目标）：列出只能运行在SPSS Modeler Server 上，而不能运行在Hadoop上的所有算法
    - 在Analytic Service 上运行（启用分割）：列出只能在Hadoop上运行，并且在上游“类型”节点对字段设置“分割” 拆分的算法
    - 在Analytic Service上运行（极大型数据）：列出只能在Hadoop上运行的所有算法

- 放弃

  ![image-20230129102146195](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129102146195.png)

- 设置

  ![image-20230129102210439](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129102210439.png)

  - 过滤出整体模型生成的字段：直接使用“自动分类”生成的模型，预览或者使用“表格”查看的时候会生成两个字段，一列是预测结果，一列是预测结果的置信度，这个结果根据下方选择的“整体方法”标准来生成的

  - 标志目标

    - 整体方法：如果目标是标志型，可以使用3个模型组合的结果作为最终预测结果，组合的标准也就是这个理的整体方法标准，包含 投票，置信度加权投票，最初倾向加权投票，最高置信度当选，平均原始倾向

      

- 注解

![image-20230129102733699](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129102733699.png)

![image-20230129102827021](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129102827021.png)



![image-20230129142058584](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129142058584.png)

![image-20230129102902977](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129102902977.png)

## 自动聚类

自动聚类会将SPSS Modeler 里面所有的聚类算法都包含其中，运行的时候会按照设置选择聚类算法，并按照一定标准筛选出最优的3个模型结果生成。

### 分析步骤

![image-20230129134130935](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129134130935.png)

#### 数据源节点

![image-20230129134153218](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129134153218.png)

#### 类型节点

![image-20230129134204671](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129134204671.png)

#### 自动聚类节点

![image-20230129134216040](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129134216040.png)

- 字段

- 模型

  ![image-20230129134744626](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129134744626.png)

  - 模型名称
  - 使用分区数据
  - 模型排序依据
    - 轮廓：该指标是评估聚类好坏的标准，数值介于0-1，越接近1越好。
    - 聚类数：模型中的聚类数
    - 最小聚类大小：最小聚类的大小
    - 最大聚类大小：最大聚类的大小
    - 最小/最大聚类大小：最小聚类与最大聚类的大小比率
    - 评估字段的重要性：评估字段的重要性
  - 模型排序方式
    - 训练分区
    - 测试分区
  - 要保留的模型数量：默认是保留3个最优的模型，在SPSS Modeler中一共就3个聚类算法

- 专家

  ![image-20230129134841644](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129134841644.png)

  数据量比较大的时候，可以在停止规则中设置规则。

- 放弃

  ![image-20230129135029866](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129135029866.png)

- 注解

![image-20230129135136535](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129135136535.png)

![image-20230129135237483](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129135237483.png)

![image-20230129135211372](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129135211372.png)

![image-20230129135508672](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129135508672.png)

![image-20230129135305543](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129135305543.png)

## 自动数值

自动数值是针对数值型指标的预测，会将SPSS Modeler里面所有用于数值型预测的算法都包含在内（不包含时间序列算法），运行的时候，会按照设置运行选择的算法，并按照一定的标准筛选出最优的3个模型

### 分析步骤

![image-20230129140306255](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140306255.png)

#### 数据源节点

![image-20230129140418028](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140418028.png)

#### 类型节点

![image-20230129140435880](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140435880.png)

#### 自动数值节点

![image-20230129140448153](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140448153.png)

- 字段

- 模型

  ![image-20230129140535275](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140535275.png)

  - 模型名称
  - 使用分区数据
  - 为每个分割构建模型
  - 模型排序依据
    - 相关性：计算每个输入指标与目标的Person相关系数，数值在-1~1，绝对值越接近1说明相关性越强
    - 字段数：用户构建模型的字段数，字段数越少，模型越简化
    - 相对误差：相对误差是模型实际值相对于预测值的方差与实际值相对于平均值的比率，误差越小，模型越好
  - 模型排序依据
    - 训练分区
    - 测试分区

- 专家

  ![image-20230129140700497](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140700497.png)

  - kk

- 设置

  ![image-20230129140722066](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129140722066.png)

- 注解

![image-20230129141217125](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141217125.png)

![image-20230129141648392](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141648392.png)

![image-20230129141709939](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141709939.png)

![image-20230129141442492](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141442492.png)

![image-20230129141823556](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141823556.png)

![image-20230129141842824](http://cdn.luluwanlong.cn/spss-modeler-other-analysis-image-20230129141842824.png)

