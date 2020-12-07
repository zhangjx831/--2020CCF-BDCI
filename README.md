# Traffic condition prediction

近期参加了中国计算机学会举办的道路状况时空预测比赛

比赛链接：https://www.datafountain.cn/competitions/466

最终我们组的比赛成绩是29/2482，在有成绩的组中排名29/214

Github的文件分别是我们构建的模型

HA.py: 历史平均方法

ARIMA.py: 时间序列ARIMA方法

LSTM.py: 时间序列LSTM方法

lightgbm.py: 集成学习LightGBM方法和决策树方法

下面从数据预处理、特征工程、模型构建、调参四个方面来介绍我们的工作

## 数据预处理

原始数据中主要包括四部分

attr.txt: 原始道路特征，主要包括道路id，长度，宽度，方向等特征

topo.txt: 道路拓扑特征，包括道路id和每个道路的下游道路

201907xx.txt: 道路数据，包括现在的时间段、待预测的时间段、近5个时间片的平均车速、车流量、道路特征以及前四周的5个时间片的数据等

20190801_testdata.txt: 测试集数据，与道路数据格式相同

处理过程

1.导入道路属性数据并对字符串进行切分变为pandas DataFrame

2.导入道路拓扑属性并用两个字典inflow outflow记录道路的上下游连接信息，前两部分在loaddata()函数中实现

3.循环导入道路数据，并对数据字符串进行切分，并join道路属性数据构成整体的数据集

通过对原始数据的描述性分析，发现道路状况为1的数据占大部分80%，道路状况为2和3的占小部分，存在数据不平衡问题待之后处理

## 特征工程

原始数据的特征主要用四个，分别是不同时间片的路径平均速度、eta速度、道路状况、道路车辆数

我们首先求出待预测时间片的道路状况和所有其他特征的相关系数，发现相关系数较大的特征是各个时间片的eta速度和道路状况，之后可以通过相关系数和chi2检验进行特征筛选

除此之外，我们还根据domain knowledge构建了新的特征，主要是道路车辆数除以车道数除以道路长度，这个可以视作单位时间单位车道的车辆

由于原始数据存在类别不平衡问题，我们对label为1的数据进行欠采样，选取原始label为1数据的1/4纳入训练集

## 模型构建

我们主要构建了五种模型

1.历史平均法

原始数据集中包括前7天、14天、21天、28天同一时间段的道路状况，我们通过这四个时间片的众数预测待预测时间片的道路状况，最终这种方法的加权F1得分是0.37

2.决策树

我们根据在特征工程部分选出的特征构建决策树模型，通过这些特征预测道路状况，最终构建的决策树模型加权F1得分是0.36

3.LightGBM模型

LightGBM模型是一种基于梯度提升算法的集成学习模型，与决策树模型类似，最终构建的模型加权F1得分为0.45

4.ARIMA模型

原始的数据是时空数据，我们希望构建ARIMA模型来对时间序列建模，最终ARIMA模型的加权F1得分为0.29

5.LSTM模型

与ARIMA模型类似，我们也希望对时间序列进行建模，选择了RNN循环神经网络中的LSTM长短期模型，最终模型的得分为0.29

## 调参

从上面的五种模型我们可以看出，预测最准的模型是LightGBM集成学习模型，我们通过对LightGBM模型进行调参，调参采用网格搜索的方式进行，3折交叉验证，加权f1得分做为评价指标

（1）首先设置好学习率和迭代次数，学习率为0.1，迭代次数为200，加速迭代

（2）对树的深度和叶子节点数目进行调参，树的深度设置为[20, 40, 80]，叶子节点数目设置为[320, 640, 1280]，最终学习得到的最佳树深度为40，最佳叶子节点数目为1280

（3）对最小子节点样本数和权重进行调参，最终得到的最佳参数分别是40和0

（4）对特征比例进行调参，特征比例指的是每次训练选用的特征占总特征的比例，最终得到的最佳参数是1

（5）对bagging比例和bagging频率进行调参，得到的最佳参数是0.8和1

（6）对l1和l2正则化参数进行调参，最终得到的最优参数分别是0和0.2

（7）对cat_smooth每个类别拥有最小的个数进行调参，最终得到的参数是0

通过模型调参后，调小学习率，增大迭代次数，最终模型得分提高了0.02左右，达到了0.47

## 总结

（1）从模型构建的结果看，LightGBM模型效果最好，在以往大部分数据竞赛中，基于梯度提升的集成学习方法LightGBM都表现的较好。而基于时间序列的数据表现不好，这可能是因为时间序列比较短，而且待预测的时间片和当前的时间片不连续，导致预测效果不好

（2）交通模型是时空数据，LightGBM是传统的基于树的分类模型，没有考虑到原始数据集的空间拓扑结构和时间序列数据，这部分数据通常对预测结果有较大帮助，目前学界在交通客流预测领域使用较多的模型主要是T-GCN，即图卷积神经网络+时间序列模型以及他的变种，例如在空间部分采用GAT，时间部分采用LSTM并加入注意力机制等

（3）本次的数据集中的拓扑数据是下游路径id，在图中表示为有向图，针对有向图我们通常可以采用图注意力机制(GAT)模型，但是由于这次的数据时间不同步，且路网拓扑较大，所以我们没有针对路网拓扑进行建模

