<font face="黑体" size=4>
第四课的学习有以下三个重点：  
	
# 1 NLP  
&emsp;文本分类（Text Classification）是自然语言处理领域很普遍的应用，可以用来完成垃圾邮件过滤、虚假新闻识别、医疗报告诊断、用户情感分类等任务。
## 1.1 IMDb电影评论情绪分析实践  
&emsp;假如有25000条数据，每条数据包括电影评论以及评论的情绪分类（1表示喜欢，0表示不喜欢）。在分析评论情绪时，需要对评论所属语种的语言基础和习惯有一定的了解，如果神经网络初始化权值都是随机设定的，25000条数据不足以生成一个效果很好的分类器。因此我们尝试使用迁移学习来获取更多与语言相关的信息，提升模型效果。<br>
&emsp;在NLP问题的迁移学习中使用语言模型作为预训练模型，fastai创建了Language Model Zoo给不同语种和不同知识领域建立语言模型，在实践中我们将使用语言模型(a language model)作为预训练模型（为表述方便，下面提及的语言模型都是指英语领域内的语言模型）。语言模型“学习”了很多有关语言表达、世界运作的知识。给定一个句子的前面部分，语言模型可以预测后一个单词，在下面的例子中我们将使用从Wikipedia（维基百科）训练的模型作为预训练模型，为IMDB语言模型提供必要的语言知识和常识。<br>   
&emsp;Wikitext 103是Stephen Merity和他的同事共同建立的数据集，取Wikipedia篇幅最大文章的一部分后经预处理得到。在Wikitext 103基础上建立神经网络模型并通过训练不断调整权重后，模型在预测Wikipedia句子中下一个单词问题上有很好的效果。
&emsp;预训练模型生成IMDB语言模型需要FineTuning（微调），预训练模型“学习”影评写法是一个自监督学习过程（self supervised learning），自监督学习属于监督学习，但它的标签不是人工产生的，而是从输入数据中得到的。
**整体步骤：获取Wikitext103预训练模型-finetune>IMDB语言模型->IMDB分类器**
### 1.1.1文本预处理，创建TextDataBunch
#### Tokenization and Numericalization
&emsp;在预处理文本数据中，需要用到**“分词Tokenization”和“数字化Numericalization”**。
  * 分词：将句子根据“空格”、“连接词”、“标点”分割为标识符token，以“xx”开头的token为未知token。
  * 数字化：用数字表示token。 <br>
Vocab在fastai中是一个类，保存数字和token之间的对应关系。为了避免神经网络中权重矩阵太大，需要限制vocab的大小，默认vocab中不多于60000个词。如果一个词出现次数小于三次，该词不会进入vocab中。 比如用“xxunk”这一token表示出现次数小于三次的单词。<br>
其他未知token包括：<br>
xxfld：用来区分文本中的不同结构，比如概述（summary）、摘要（abstract）、正文（body）每种结构在文章中有着单独的区域，因此会被编号加以标记。如xxfld 1，xxfld 2；<br>
xxup：表示下一个单词在原文本中是全大写的。文本中有全大写的单词时，分词时将大写全部转为小写并在前面加xxup。
#### Create databunch
&emsp;使用data block API创建DataBunch有很多方式，需要根据数据类型、源文件类型等选择函数：<br>
&emsp;Where are the inputs and how to create them?（输入数据可能来自文件夹、csv表格或dataframe）<br
&emsp;How to split the data into a training and validation sets?（训练集、验证集可随机划分、通过某种索引方式划分或根据数据所在文件夹划分）<br>
&emsp;How to label the inputs?（label可能从csv文件或dataframe中获得，也可能从文件夹或某个特定函数中获得）<br>
&emsp;What transforms to apply?（可选择是否做数据增强）<br>
&emsp;How to add a test set?（可选择是否添加测试集）<br>
&emsp;How to wrap in dataloaders and create the DataBunch?
（定义创建DataBunch的一些必要参数如batch size（批大小），collate function（将样本打包成批数据的函数）等）<br>

### 1.1.2模型训练
&emsp;下面以Wikitext 103模型为预处理模型，再在IMDB整体数据集上做微调。具体模型训练步骤见jupyder文件。
#### 训练中常见问题
&emsp;（1）验证集比例设计成总体10%的技巧：<br>
&emsp;训练过程中虽然不能使用测试集的标签，但是可以用测试集中的文本来微调语言模型。在训练语言模型时可以将训练集和测试集拼接在一起，分出小部分作为验证集，从而有更多的数据可用作训练集。如果当前只有一小部分标签数据或者在做Kaggle上NLP的任务时，可以把所有的text文本数据都用来训练语言模型。<br>
&emsp;（2）模型训练时为什么每次要先fit_one_cycle(1 )迭代一次，调整最后几层的参数，再unfreeze，fit_one_cycle（）迭代更多次？<br>
&emsp;learn.freeze()是为了避免被训练好的权重再被修改，而从fastai库中引用预训练模型时，预训练模型的权重默认就是’frozen’状态的，因此直接用fit_one_cycle()训练即可，运行learn.summary()可以看到’frozen’状态的层的属性’requires_grad = False’,梯度不会在这些层上计算。
一个神经网络最开始的几层大多数情况下用来理解一些低级的基础特征，比如图像分类问题中识别曲线、直线等，通过大规模数据的训练，预训练模型的前几层已经能很好识别这些特征。后面几层通常获取特定的数据集更高级、复杂的特征，通过全连接网络将前面层的输出结合起来再应用到现有的数据集上。<br>

#### 训练模型的一般步骤
（1）load the model   <br>
（2）freeze the initial layers<br>
 Learn.freeze()<br>
（3）training model<br>
 在fit_one_cycle()之前用lr_find() 寻找最佳学习率。Learn.fit_one_cycle()，在模型结构中有很多layer groups，仅仅训练最后一个layer group，也就是全连接网络，不要过多训练以防过拟合。<br>
（4）unfreeze the layers<br>   
 Learn.unfreeze()<br>   
 所有层都可训练(requires_grad=True)<br>
（5）training model   <br>
 Learn.fit_one_cycle(2,max_lr = slice(le-6,le-4))，最初几层学习率要设置比较小，因为前面几层权重不需要调整太多；因此随着层加深，学习率会逐渐提高，尤其全连接层需要更多的“微调”。<br>   
（6）save model parameters   <br>
 Learn.save()   <br>

### 1.2情绪分析算法—ULMFiT
&emsp;我们进行以上操作遵循的是ULMFiT(Universal Language Model Fine-tuning for Text Classification)的思路，ULMFiT（基于微调的通用语言模型）是Jeremy Howard和Sebastian Ruder在2018年发表的一篇论文中提出来的，在六个文本分类任务中取得显著效果。<br> 
&emsp;ULMFiT方法包括三个阶段：  <br>
&emsp;1、General-domain LM pretraining   <br>
&emsp;首先在wiki上预训练语言模型，从而学到大规模语料的语法特性。   <br>
&emsp;2、Target task LM ﬁne-tuning   <br>
&emsp;在目标数据集上fine-tuning语言模型，以达到适应目标数据集的目的。<br>
&emsp;在这一步中使用了两个trick：<br>
&emsp;（1）Discriminative fine-tuning<br>
&emsp;为每一层设置不同的学习率，用SGD（随机梯度下降法）在第t步更新参数时的公式为：<br> 
&emsp;其中θ^l表示第L层的参数，η^l表示第L层的学习率，∇(_θ^ )J(θ)表示目标函数的梯度。<br>
&emsp;在实际中先微调最后一层选择最后一层的学习率η^L，再通过公式<br>
η^(l-1)=η^l/2.6
&emsp;计算前一层的学习率。（与实践中learn.fit_one_cycle()学习率区间选取相对应）
（2）Slanted triangular learning rates
&emsp;倾斜三角学习率（STLR）使模型在训练过程中先线性增加学习率，再线性降低。即层之间学习率不同，在训练过程中同一层的学习率也会根据STLR有所变化。
 
（3）Target task classiﬁer ﬁne-tuning
&emsp;在微调模型用来完成目标文本分类任务时，在预训练语言模型后增加两个线性模块。每个模块包括batch normalization、dropout、ReLU activation（中间层使用）、Softmax activation（最后一层输出类别的概率分布）。
&emsp;这一步中使用了四个 trick：
&emsp;（1）Concat pooling
长文本分类任务如果仅仅考虑模型最后一个time step中隐藏状态h_T可能会丢失数据，因此将h_T和GPU内存中存储的尽可能多的time steps隐藏状态的最大池化表示和平均池化表示连接起来：h_c = [h_T , maxpool(H), meanpool(H)]
&emsp;（2）Gradual unfreezing
从最后一层开始每次训练只unfreeze一层，微调当前所有的’unfrozen’层。
&emsp;（3）BPTT for Text Classification (BPT3C)
&emsp;（4）Bidirectional language model
预训练一个前向语言模型和后向语言模型，使用BPT3C分别微调生成文本分类器，最后将两个文本分类器的结果做平均。
# 2、处理表格数据
&emsp;表格数据分析在很多方面有所应用,如诈骗检测、销量预测等。人们经常使用逻辑回归（logistic regression）、随机森林（random forests），梯度提升树（gradient boosting machines）来分析表格数据，接下来我们通过神经网络来分析，主要应用fastai.tabular库。<br>
&emsp;在fastai中如何将NLP中标签化的数据和metadata元数据（表格数据）结合起来，比如用演员、制作年份等来完成IMDB分类呢？和结合类别型变量和连续型变量类似，在神经网络中可以将多种输入在某一层结合在一起。比如用RNN处理文本数据，用CNN处理图像数据，用tabular模型处理metadata元数据，之后再用某种方法组合在一起，经过全连接网络就能实现端对端的训练。<br>
&emsp;表格数据的时间序列不需要用RNN来处理，只用提取一些特殊的列，比如一周的第几天，是否是周末，是否是假期等等。
## 2.1 区分类别与连续变量 
&emsp;值为连续性数字的变量被称为连续变量，连续变量可以为任意的数字，有无限个可能的值，比如人可以是13.36岁或19.4岁；值只能从离散的集合中选取的变量称为类别变量，如婚姻状况变量，只能有已婚、未婚、离异等几个特定选项、职位变量可选项有很多，但同样需要从一个集合中选取取值。在神经网络中处理类别变量和连续变量的方法是不同的。<br>
## 2.2 fastai.tabular模块的使用
详见jupyder文件
# 3、Collaborative filtering协同过滤模型
## 3.1协同过滤模型简介
&emsp;协同过滤模型表示用户喜欢哪些商品或购买了哪些商品。在最基础的协同过滤模型中，数据只有两列，比如用useId和movieId的对应关系表示用户看了哪部电影。Amazon通过记录userId和productId的对应关系记录用户买了哪些商品，此外还可以记录用户的购买时间、评论内容、评论时间等。<br>
有两种方式可以表示协同过滤结构:<br>
* two-column方法，即用列表示所有的对应元素，userId-movieId每个对应关系表示某个用户看了某个电影。
 
* 是用行表示userId，列表示movieId，对应矩阵中M[userId][movieId]的值为1表示该用户看了id为movieId的电影，值也可以是用户的评分。
 
&emsp;因为大部分用户没有看过大部分电影，数据用第二种方法直接表示将产生一个巨大的稀疏矩阵，存储效率不高。因此协同过滤结构的数据一般用two-column方法或用特殊稀疏矩阵格式存储。
## 3.2 Cold-start problem
&emsp;协同过滤要在具备了有关商品、用户的一些信息之后才能进行。在协同过滤的实际应用中，经常会遇到“cold start problem”，指给新用户推荐电影，或把新电影推荐给用户时因为协同过滤系统缺少数据造成的问题。
&emsp;Fastai还没有解决“cold start problem”，这个问题概念上只能通过新增元数据驱动模型（metadata driven model）来解决。Netflix通过UX(user experience)来解决cold start problem，新用户登录时Netflix会询问用户是否看过或者喜欢某些电影，根据用户的回答再给一批电影让用户选择，由此为协同过滤收集足够的数据。<br>
&emsp;如果在实际应用中不便询问用户，可以使用基于元数据的表格模型(metadata based tabular model)来解决cold-start problem，比如通过用户家乡、年龄、性别给出一系列基础的推荐。<br>

## 3.3 用电子表格完成协同过滤
* [ctrl/command]+右箭头：跳到表中当前行的最右端单元格
* [ctrl/command]+右箭头+下箭头：跳到表的右下角
* [ctrl/command]+左箭头+上箭头：跳到表的左上角

&emsp;数据源：MovieLens中看最多电影的userId和被观看次数最多的movieId作为协同过滤矩阵的行和列。
（1）每个用户和每部电影都设置5个随机数，如用户设置U=(U1,U2,U3,U4,U5)，电影设置M=(M1,M2,M3,M4,M5)T <br>
（2）设计一个线性模型：预测用户给电影的评分时让U和M点乘得到评分结果。这是神经网络的起点，也就是矩阵点乘。<br>
（3）设计误差函数MSE或RMSE<br>
误差求和：S = ∑▒〖(TrueRating-PredictedRating)2〗（EXCEL中用SUMXMY2函数计算）
计算MSE = sqrt(S/count)
（4）Excel Solver：用梯度下降法调整权重矩阵降低误差。Excel中gradient descent solver的名称为“Solver”，Data->Solver弹出设置页面，设置目标函数（误差函数）所在的cell以及所有变量所在的cell，设置目标函数取最小值，点击Solve可以看到误差函数的值在下降。<br>
&emsp;基本上这就简单创建了一个神经网络，可以利用线性模型和梯度下降解决协同过滤问题。 
## 3.4 用编辑器查看fastai的源代码
1、找到要查看的类名或函数名<br>
2、跳到代码中类或函数的实现部分<br>
&emsp;比如collab_learner函数创建了EmbeddingDotBias模型，EmbeddingDotBias模型包括__init__()函数和forward()函数，数据包括用户和商品对应的weights和bias。<br>
## 3.5 Embedding Matrix和bias
&emsp;Embedding 是将离散变量用连续向量表示的一个方式。One-hot编码是另外一种最常见的表示离散变量的方法，首先计算出要表示的类别变量的总个数N， 每个变量用N-1个0和单个1组成的vector来表示每个类别，但是One-hot编码有两个很明显的缺点：
1、对于非常多类型的类别变量，变换后的向量维数过于巨大且过于稀疏
2、映射之间完全独立，不能表示出不同类别之间的关系。
&emsp;为了解决这两个问题，我们采用embedding用较少的维度表示出每个类别，并且还能表现出不同类别之间的关系。Embedding可以通过监督学习网络在训练中逐渐改善。<br>
&emsp;在userId-movieId这个例子中， Embedding Matrix就是user和movie的权重矩阵，每个user或每个movie对应的向量也可以称为Embedding Matrix，可以点乘得到预测的评分结果。<br>
在实际中存在一些电影会受更多人喜爱，同时一些人会更加喜欢看电影，因此在点乘之后，增加bias表示电影的受欢迎程度和人们喜欢看电影的程度。 
参考：
https://www.kaggle.com/hortonhearsafoo/fast-ai-v3-lesson-3-imdb
fastai官方文档：https://docs.fast.ai
embedding: https://www.cnblogs.com/jamnoble/p/11588241.html
ULMFit：https://arxiv.org/pdf/1801.06146.pdf
《ULMFiT》 阅读笔记：https://zhuanlan.zhihu.com/p/47344283
https://blog.csdn.net/triplemeng/article/details/82828480
</font>
