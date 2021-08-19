---
title: 通过pyspark方式使用xgboost
description: 通过pyspark方式使用xgboost
categories:
 - 机器学习
tags:
 - pyspark
 - xgboost
---

# 通过pyspark方式使用xgboost


## 背景

我们想利用python-spark 从hive中读取特征数据， 调用xgboost算法，进行模型计算。

因为spark会经过yarn调度在hadoop的每个节点进行分布式计算， 而 sklearn是单机版, 因此无法使用sklearn包中的算法集。
我们选用pyspark包， 其中pyspark.ml子包有丰富的算法集，且支持分布式计算，比如有kbeans, 逻辑回归等， 但没有我们这里所说的xgboost算法。

## pyspark中引入xgboost

java中有成熟的xgboost包， 并且可以运行在spark中， 所以可以考虑python代码调用java包中的xgboost方法， 也算是曲线救国。

因此我们需要以下包:

1. xgboost4j.jar : java的xgboost4j.jar包 下载地址：https://repo1.maven.org/maven2/ml/dmlc/
2. xgboost4j-spark： java的spark上使用的xgboost4j-spark ,下载地址: https://repo1.maven.org/maven2/ml/dmlc/,
3. sparkxgb.zip: python打包的代码，封装java调用

因为无法直接调用java包， 所以我们用python包装一层适配代码。
调用链路： 业务代码—>sparkxgb—> java包

sparkxgb包并没有发布到pip仓库中， 代码仓库为: https://github.com/sllynn/spark-xgboost  
我们可以下载sparkxgb最新release代码: https://github.com/sllynn/spark-xgboost/archive/refs/tags/v0.9.zip

```
#!/usr/bin/env python
# -*- coding:utf8 -*-


"""
-------------------------------------------------
   Description :  打分是否被欺骗
-------------------------------------------------
"""


import os
import sys
import time
import pandas as pd
import numpy as np
from pyspark import SparkConf, SparkContext
import pyspark.sql.types as typ
import pyspark.ml.feature as ft
from pyspark.sql.functions import isnan, isnull,col
import pyspark
from pyspark.sql.session import SparkSession
from pyspark.sql.types import *
from pyspark.ml.feature import StringIndexer, VectorAssembler
from pyspark.ml.evaluation import BinaryClassificationEvaluator,MulticlassClassificationEvaluator
from pyspark.ml import Pipeline
from sparkxgb import XGBoostClassifier
import argparse




# 命令行参数
parser = argparse.ArgumentParser(description='user_rating_cheat')
parser.add_argument('--dt', type=str, required=True, help='要分析的数据日期')
args = parser.parse_args()


# 日期
dt = args.dt


# spark会话
spark = SparkSession.builder.appName('user_rating_cheat').enableHiveSupport().getOrCreate()


# 查询打分日志表
sql = """
select
    id,user_id,channel_id, article_id, if(rating=1,1,0) label, rating, msg, status, ip, user_agent, creation_date
from safe.dwd_fact_userlog_article_rating_log where dt='{dt}' ;
""".format(dt=dt)


# 查询数据
rating_log_df = spark.sql(sql)


"""
# 打印展示
rating_log_df.show(truncate=False)
+--------+--------+----------+----------+-----+------+---+------+-------------+----------------------------------------------------------------------------------------------+-------------------+
|id      |user_id |channel_id|article_id|label|rating|msg|status|ip           |user_agent                                                                                    |creation_date      |
+--------+--------+----------+----------+-----+------+---+------+-------------+----------------------------------------------------------------------------------------------+-------------------+
|83027754|5178880 |3         |21434324  |1    |1     |   |0     |60.247.60.179|smzdm_android_V9.9.6.29 rv:676 (1809-A01;Android8.1.0;zh)smzdmapp|android@9.9.6.29            |2020-12-21 14:10:50|
|83027755|5178880 |3         |21451966  |1    |1     |   |0     |60.247.60.179|smzdm_android_V9.9.6.29 rv:676 (1809-A01;Android8.1.0;zh)smzdmapp|android@9.9.6.29            |2020-12-21 14:13:05|
|83027756|11238222|80        |70073612  |1    |1     |   |0     |223.71.0.228 |smzdm 9.9.9.11 rv:94 (iPhone XS Max; iOS 14.0; zh_CN)/iphone_smzdmapp/9.9.9.11|iphone@9.9.9.11|2020-12-21 14:13:05|
"""


# 将字符串特征转换为向量特征
ipIndexer = StringIndexer()\
  .setInputCol("ip")\
  .setOutputCol("ip_index")\
  .setHandleInvalid("keep")


vectorAssembler = VectorAssembler()\
  .setInputCols(["user_id", "channel_id", "article_id", "ip_index"])\
  .setOutputCol("features")


# 定义分类器，调优参数，需修改sparkxgb.zip中参数
xgboost = XGBoostClassifier(
    featuresCol="features",
    labelCol="label",
    predictionCol="prediction",
    missing = 0.0
)


# 构建执行pipeline阶段
pipeline = Pipeline(stages=[ipIndexer, vectorAssembler, xgboost])


# 切割出训练集、测试集
trainDF, testDF = rating_log_df.randomSplit([0.8, 0.2], seed=24)


# 打印输出训练集
trainDF.show(2)


# 通过训练集，训练模型
model=pipeline.fit(trainDF)


# 预测测试集
result = model.transform(testDF).select(col("id"), col("label"), col('probability'), col("prediction"), col("rawPrediction"))


#打印预测结果
result.show()
"""
+--------+-----+--------------------+----------+
|      id|label|         probability|prediction|
+--------+-----+--------------------+----------+
|83027742|    1|[0.36363637447357...|       1.0|
|83027748|    0|[0.39999997615814...|       1.0|
|83027753|    1|[0.60000002384185...|       0.0|
|83027755|    1|[0.39999997615814...|       1.0|
+--------+-----+--------------------+----------+
"""


# # 转换为pandas格式处理
# result = pre_df.toPandas()


# 使用混淆矩阵评估模型性能[[TP,FN],[TN,FP]]
TP = result.filter(result['prediction'] == 1).filter(result['label'] == 1).count()
FN = result.filter(result['prediction'] == 0).filter(result['label'] == 1).count()
TN = result.filter(result['prediction'] == 0).filter(result['label'] == 0).count()
FP = result.filter(result['prediction'] == 1).filter(result['label'] == 0).count()


# 计算查准率 TP/（TP+FP）
precision = TP/(TP+FP)
# 计算查全率 TP/（TP+FN）
recall = TP/(TP+FN)
# 计算F1值 （TP+TN)/(TP+TN+FP+FN)
F1 =(2 * precision * recall)/(precision + recall)
print("The 查准率 is :",precision)
print("The 查全率 is :",recall)
print('The F1 is :',F1)
# AUC为roc曲线下的面积，AUC越接近与1.0说明检测方法的真实性越高
auc = BinaryClassificationEvaluator(labelCol='label').evaluate(result)
print("The auc分数 is :",auc)


"""
The 查准率 is : 0.6666666666666666
The 查全率 is : 0.6666666666666666
The F1 is : 0.6666666666666666
The auc分数 is : 0.5
"""


#TODO 模型输出到文件， 供后面使用
```

我们的启动脚本如下:

```
#!/bin/bash


dt=$1


cd src && zip -r src.zip * && mv src.zip .. && cd -


# --master yarn ：运行到yarn集群，固定写法
# --deploy-mode cluster：AM运行到yarn中，如果改成client则需要确保本地目录有./Python/bin/python3
# --num-executors 1：一个executor容器
# --archives hdfs:///Python.zip#Python：从hdfs集群下载/Python.zip到executor工作目录，并解压到Python目录
# --py-files ./src.zip：项目python源代码，会解压到executor的某目录下并令PYTHONPATH指向该目录
# --conf spark.pyspark.python=./Python/bin/python3：指定使用自行上传的Python
# bash run.sh 2020-12-21
spark-submit \
  --conf spark.pyspark.python=./Python/bin/python3 \
  --master yarn \
  --deploy-mode client \
  --num-executors 32 \
  --executor-memory 4G \
  --archives hdfs:///warehouse/safe/utils/Python.zip#Python \
  --jars ./resources/xgboost4j-spark_2.12-1.0.0.jar,./resources/xgboost4j_2.12-1.0.0.jar \
  --py-files ./src.zip,./resources/sparkxgb.zip \
  src/main.py --dt $dt
```

即然需要使用上面三个离线的jar包，和python包， 我们则需要在spark-submit阶段，将包加入到系统环境中

所以需要--jars 引入jar包， —py-files引入sparkxgb包。我们的开发目录如下:

```
├── resources
│   ├── sparkxgb.zip
│   ├── xgboost4j-spark_2.12-1.0.0.jar
│   └── xgboost4j_2.12-1.0.0.jar
├── run.sh
└── src
    └── main.py
```

我们执行代码，发现报错，和我们预期不符

```bash run.sh 2020-12-21```

目前报错：

```
py4j.protocol.Py4JJavaError: An error occurred while calling None.ml.dmlc.xgboost4j.scala.spark.XGBoostClassifier.
...
Caused by: java.lang.ClassNotFoundException: scala.Product$class
```

其他的报错:

```
Traceback (most recent call last):
  File "/data/workspace/users/litong/python/user_rating_cheat/src/main.py", line 83, in <module>
    xgboost = XGBoostClassifier(
  File "/usr/local/service/spark/python/lib/pyspark.zip/pyspark/__init__.py", line 110, in wrapper
  File "/data/workspace/users/litong/python/user_rating_cheat/resources/sparkxgb.zip/sparkxgb/xgboost.py", line 85, in __init__
  File "/data/workspace/users/litong/python/user_rating_cheat/resources/sparkxgb.zip/sparkxgb/common.py", line 68, in __init__
  File "/usr/local/service/spark/python/lib/pyspark.zip/pyspark/ml/wrapper.py", line 69, in _new_java_obj
  File "/usr/local/service/spark/python/lib/py4j-0.10.9-src.zip/py4j/java_gateway.py", line 1568, in __call__
  File "/usr/local/service/spark/python/lib/pyspark.zip/pyspark/sql/utils.py", line 131, in deco
  File "/usr/local/service/spark/python/lib/py4j-0.10.9-src.zip/py4j/protocol.py", line 326, in get_return_value
py4j.protocol.Py4JJavaError: An error occurred while calling None.ml.dmlc.xgboost4j.scala.spark.XGBoostClassifier.
: java.lang.NoClassDefFoundError: org/apache/spark/ml/param/shared/HasWeightCol$class
    at ml.dmlc.xgboost4j.scala.spark.XGBoostClassifier.<init>(XGBoostClassifier.scala:43)
    at ml.dmlc.xgboost4j.scala.spark.XGBoostClassifier.<init>(XGBoostClassifier.scala:48)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
    at py4j.reflection.MethodInvoker.invoke(MethodInvoker.java:247)
    at py4j.reflection.ReflectionEngine.invoke(ReflectionEngine.java:357)
    at py4j.Gateway.invoke(Gateway.java:238)
    at py4j.commands.ConstructorCommand.invokeConstructor(ConstructorCommand.java:80)
    at py4j.commands.ConstructorCommand.execute(ConstructorCommand.java:69)
    at py4j.GatewayConnection.run(GatewayConnection.java:238)
    at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.ClassNotFoundException: org.apache.spark.ml.param.shared.HasWeightCol$class
    at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
```

在这里阻塞了很长时间，也查询了很多资料， 但网上讲的不够透彻， 核心问题是：
spark的版本和内在依赖scala版本 需要和 xgboost4j.jar 和xgboost4j-spark.jar包依赖的版本保持一致！！！！
查看spark版本:


```
> spark-shell

输出:
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.0.0
      /_/


Using Scala version 2.12.10 (OpenJDK 64-Bit Server VM, Java 1.8.0_252)
Type in expressions to have them evaluated.
Type :help for more information.
```

我们再进入到xgboot4j下载页面(https://repo1.maven.org/maven2/ml/dmlc/)

```
mxnet/                                                           -         -      
xgboost-jvm/                                                     -         -      
xgboost-jvm_2.11/                                                -         -      
xgboost-jvm_2.12/                                                -         -      
xgboost4j/                                                       -         -      
xgboost4j-example/                                               -         -      
xgboost4j-example_2.11/                                          -         -      
xgboost4j-example_2.12/                                          -         -      
xgboost4j-flink/                                                 -         -      
xgboost4j-flink_2.11/                                            -         -      
xgboost4j-flink_2.12/                                            -         -      
xgboost4j-gpu_2.12/                                              -         -      
xgboost4j-spark/                                                 -         -      
xgboost4j-spark-gpu_2.12/                                        -         -      
xgboost4j-spark_2.11/                                            -         -      
xgboost4j-spark_2.12/                                            -         -      
xgboost4j_2.11/                                                  -         -      
xgboost4j_2.12/                                                  -         -      
```

发现下载目录的后缀，即为Scala版本号， 所以这里需要下载和Spark-Scala版本下载一致的即可, 不一致的话，会有各种诡异的兼容问题。

参考文章

1. pyspark 数据分析-xgboost : https://zhuanlan.zhihu.com/p/101997567
2. sparkxgb包源码： https://github.com/sllynn/spark-xgboost
3. xgboot包下载地址: https://repo1.maven.org/maven2/ml/dmlc/
4. 主要参考: xgboost的分布式版本(pyspark)使用测试: https://zhuanlan.zhihu.com/p/139722007
5. 老版本的实现方式： https://towardsdatascience.com/pyspark-and-xgboost-integration-tested-on-the-kaggle-titanic-dataset-4e75a568bdb
6. 源码方式: https://github.com/BogdanCojocar/medium-articles/tree/master/titanic_xgboost
7. 另一个实现方式: https://zhuanlan.zhihu.com/p/273756067
8. spark版本问题: https://blog.csdn.net/zc_stats/article/details/106974862
9. 官方讨论: https://github.com/dmlc/xgboost/issues/1698
10. spark-submit提交参数说明 https://blog.csdn.net/xc_zhou/article/details/118617600
11. spark模型评估 https://blog.csdn.net/yawei_liu1688/article/details/111881592


