---
title: 贝叶斯机器分类实践评论分类
description: 贝叶斯机器分类实践评论分类
categories:
 - 机器分类
tags:
 - 机器分类
 - 贝叶斯分类
---

评论文本基于贝叶斯分类初次实践

## 背景

去年做的一个项目，要求通过机器学习方式，对评论进行分类， 比如哪些是“人身攻击”“色情”“政治”。 经过调研最终使用 贝叶斯分类来解决问题， 后续准确率达到90%，效果还不错。 因为是首次应用机器学习项目， 再这里记录下实践过程

## 技术选型

1. 语言: python
2. 类库： sklearn， 
3. 分类器: 朴素贝叶斯分类

最开始我就想上深度学习等，但根据公司算法团队大佬推荐， 贝叶斯分类效果不错， 本来我嫌这个分类有些土， 但因为操作简单，还是先上手操作，实践为王。


## 文本分类基本流程

0. 确定分类类型+原数据准备，对评论标记合适分类类型
1. 分词， 选择合适的分词器。 比如使用jieba分词等
2. 过滤停顿词
3. 拆分评论集 为「训练集」和「测试集」
4. 对非结构化的「训练集」评论数据进行向量化处理
4. 拟合贝叶斯分类
5. 预测「测试集」，获得结果
6. 上面进行微调,

下面我们细化流程， 详细讲解。


## 确定分类数+数据准备

预先需要确定分类类型， 比如“人身攻击”，“政治”，“联系方式”。 分类不能有包含关系， 处与「并集」效果最好。

我们最终确定如下分类：

1. 正常评论
2. 联系方式
3. 人身攻击


数据准备是最繁琐的， 需要手动给评论数据打上分类类型。几千条数据还好， 几十万，几百万数据就要抓狂。 这里花费了几天的工作量。 总结出的小窍门有：

1. 关键词筛选， 比如含有“VX”“微信”字样， 90%属于联系方式， 因此批量筛选这部分数据。

## 分词

这里没有选择额外的分词器， 因为评论内容特殊， 中英文混杂， 发现分词效果都比较一般， 因此直接选择的是针对字符分词

## 过滤停顿词

有些语气助词， 比如“的”，“得”，在每个语句中都存在， 价值不高，需手动过滤替换即可

## 预先向量化处理

我理解是将非结构化的评论数据，进行结构化处理。

先科普`词袋模型`

> 词袋模型假设我们不考虑文本中词与词之间的上下文关系，仅仅只考虑所有词的权重。而权重与词在文本中出现的频率有关。

> 词袋模型首先会进行分词，在分词之后，通过统计每个词在文本中出现的次数，我们就可以得到该文本基于词的特征，如果将各个文本样本的这些词与对应的词频放在一起，就是我们常说的向量化。向量化完毕后一般也会使用 TF-IDF 进行特征的权重修正，再将特征进行标准化。 再进行一些其他的特征工程后，就可以将数据带入机器学习模型中计算。

> 词袋模型的三部曲：分词（tokenizing），统计修订词特征值（counting）与标准化（normalizing）。

> 词袋模型有很大的局限性，因为它仅仅考虑了词频，没有考虑上下文的关系，因此会丢失一部分文本的语义。


重点是不考虑<strong>上下文</strong>关系, 口袋模型需要的词频，可以使用 sklearn 中的 CountVectorizer, tfidf 来完成。

### 词频向量化

CountVectorizer 类会将文本中的词语转换为词频矩阵，例如矩阵中包含一个元素a[i][j]，它表示j词在i类文本下的词频。

通过`fit_transform`函数计算各个词语出现的次数，

通过`get_feature_names()`可获取词袋中所有文本的关键字

通过`toarray()`可看到词频矩阵的结果。 

我们进行实践操作

```python

#!/usr/bin/python
# coding: utf-8

from sklearn.feature_extraction.text import CountVectorizer

vectorizer = CountVectorizer(min_df=1)

corpus = ['This is the first document.',
          'This is the second second document.',
          'And the third one.',
          'Is this the first document?',
          ]
X = vectorizer.fit_transform(corpus)
feature_name = vectorizer.get_feature_names()

#输出结果
print (feature_name)
print (X)
print (X.toarray())

```

结果输出如下

```
['and', 'document', 'first', 'is', 'one', 'second', 'the', 'third', 'this']


  (0, 1)    1     
  (0, 2)    1
  (0, 6)    1
  (0, 3)    1
  (0, 8)    1
  (1, 5)    2
  (1, 1)    1
  (1, 6)    1
  (1, 3)    1
  (1, 8)    1
  (2, 4)    1
  (2, 7)    1
  (2, 0)    1
  (2, 6)    1
  (3, 1)    1
  (3, 2)    1
  (3, 6)    1
  (3, 3)    1
  (3, 8)    1
左边的括号中的第一个数字是文本的序号i，第2个数字是词的序号j，注意词的序号是基于所有的文档的。第三个数字就是我们的词频。

[[0 1 1 1 0 0 1 0 1]
 [0 1 0 1 0 2 1 0 1]
 [1 0 0 0 1 0 1 1 0]
 [0 1 1 1 0 0 1 0 1]]
 
 这里一共有9个词， 语句会转化为 9维向量，词顺序是固定的，每个语句按词出现的次数，来决定矩阵中元素的值。
 
 因为有些词在文本中词频较高，但是不重要， 比如 `is`, 这个时候，我们就可以用下面的TF-IDF处理
 
```

### TF-IDF处理

TF-IDF（Term Frequency–Inverse Document Frequency）是一种用于资讯检索与文本挖掘的常用加权技术。TF-IDF是一种统计方法，用以评估一个字词对于一个文件集或一个语料库中的其中一份文件的重要程度。
主要基于以下两种条件
1. 字词的重要性随着它在文件中出现的次数成正比增加，
2. 但同时会随着它在语料库中出现的频率成反比下降。

TF-IDF加权的各种形式常被搜索引擎应用，作为文件与用户查询之间相关程度的度量或评级。

TF-IDF的主要思想是：如果某个词或短语在一篇文章中出现的频率TF高，并且在其他文章中很少出现，则认为此词或者短语具有很好的类别区分能力，适合用来分类。TF-IDF实际上是：TF * IDF。

* 词频（Term Frequency，TF）指的是某一个给定的词语在该文件中出现的频率。即词w在文档d中出现的次数count(w, d)和文档d中总词数size(d)的比值。

```
    tf(w,d) = count(w, d) / size(d)
    
```



这个数字是对词数 (term count) 的归一化，以防止它偏向长的文件。（同一个词语在长文件里可能会比短文件有更高的词数，而不管该词语重要与否。）


* 逆向文件频率（Inverse Document Frequency，IDF）是一个词语普遍重要性的度量。某一特定词语的IDF，可以由总文件数目除以包含该词语之文件的数目，再将得到的商取对数得到。即文档总数n与词w所出现文件数docs(w, D)比值的对数。　

```
idf = log(n / docs(w, D))
```
如果一个词越常见，那么分母就越大，逆文档频率就越小越接近0。分母之所以要加1，是为了避免分母为0（即所有文档都不包含该词）。log表示对得到的值取对数。

## fit,fit_transform，transform区别

fit: 只计算所需的标准差，平均值， 等内部状态变量, 这个是针对全量测试数据，获取低阶全局变量， 模型会存储该步骤内容
transform: 在fit基础上，对数据进行标准化，降维，归一化等高阶操作。

fit_transform: 连接fit, transform 两步操作

我们的训练模型会持久化`fit`流程中 计算出的全局变量

对testData使用同样的均值、方差、最大最小值等指标进行转换transform(testData)，从而保证train、test处理方式相同

必须先用fit_transform(trainData)，之后再transform(testData)
如果直接transform(testData)，程序会报错
如果fit_transfrom(trainData)后，使用fit_transform(testData)而不transform(testData)，虽然也能归一化，但是两个结果不是在同一个“标准”下的，因为其，具有明显差异。(一定要避免这种情况)


## 拟合贝叶斯分类

sklearn已经封装好，贝叶斯使用方式， 我们直接调用即可

```
mnb_count_clf = MultinomialNB(alpha=1.)
mnb_count_clf.fit(X_extract_train, y_train)
```

## 对测试集进行预测，评估准确率

对应代码如下

```
accuracy_rate = metrics.accuracy_score(mnb_count_clf.predict(X_extract_test), y_test)  # 同上，作用基本一致
accuracy_num = metrics.accuracy_score(mnb_count_clf.predict(X_extract_test), y_test, normalize=False)
print("test数据准确率为:%s, 准确数是:%d, 测试样本总数:%d \n" % (str(accuracy_rate), accuracy_num, len(y_test)))

print(metrics.classification_report(mnb_count_clf.predict(X_extract_test), y_test,
                                    target_names=['class 1', 'class 2', 'class 3', 'class 4', 'class 5', 'class 6',
                                                  'class 7', 'class 8']))
```



## 保存导出模型

```
joblib.dump(mnb_count_clf, "train_model.m" + DATA_TYPE + VECTOR_TYPE)
joblib.dump(mnb_count_clf, "clf_model.m")
```


## 常规预测如下

```
count_vec = joblib.load("count_train_model.m" + DATA_TYPE + VECTOR_TYPE)
X_extract_test = count_vec.transform(data_list)

clf = joblib.load("train_model.m" + DATA_TYPE + VECTOR_TYPE)

print(clf.predict(X_extract_test[0:1]))
prob_list = clf.predict_proba(X_extract_test[0:1])[0]
print(["{0:0.4f}".format(i) for i in prob_list])
```


## 参考资料

1. https://zhuanlan.zhihu.com/p/42297868
2. https://datascience.stackexchange.com/questions/12321/difference-between-fit-and-fit-transform-in-scikit-learn-models
3. https://blog.csdn.net/m0_37324740/article/details/79411651
4. https://www.cnblogs.com/lliuye/p/9178090.html

