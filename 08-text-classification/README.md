# 实例分析：文本分类


## 准备知识

* 文本分类
* Python
* Hadoop Streaming



## 研究背景

一篇被某期刊接受的文章和一篇没有被某期刊接受的文章之间的区别在哪 儿呢?这个问题实际上是一个
文本分类的问题,对于一篇文章而言,要么被某期 刊接受,要么没有被某期刊接受,此时对于任意一篇文
章就有了一个 class 类别, 可以用 0-1 来表示,1 表示被接受,0 表示未被接受。把 class 作为因变
量进行 研究,接下来的问题是,什么因素会影响一篇是否被某期刊接受呢?因此需要找 到显著的自变量,
文本数据不像数值型数据,它是由词组成,不同的词具有不同 的权重,那么思路就可以转换为在文本中进
行特征提取。特征提取之后的数据格 式为:


    lable1 index1:featureValue1 index2:featureValue2 index3:featureValue3 ...

    lable2 index1:featureValue1 index2:featureValue2 index3:featureValue3 ...


其中,label1 和 label2 分别代表 1(被接受)和 0(未被接受),而 index1 表示 words 语料库中的词,
对应 featureValues1 则表示 index1 中词的特征值。 本文直接把特征(词语)在文本中出现的频率当
作其特征值,这样就把这个特征 值映射到[0,1]空间上了,特征值计算采取简单的频率计算公式:


    featureValues = 词语出现的次数 / n


当把文本转换成向量的形式后,利用算法对训练集进行训练得到模型,然后 计算训练集的误判率,并利用
测试集进行验证,计算测试集的误判率。

## 实践研究


现有的分类技术都已经非常成熟,SVM、KNN、Decision Tree、AN、NB 在 不同的应用中都展示出较好的
效果,由于支持向量机(Support Vector Machine) 在解决小样本、非线性及高维模式识别中表现出许多
特有的优势,并能够推广应 用到函数拟合等其他机器学习问题中,因此本文选取支持向量机算法进行文
本分类。支持向量机方法是建立在统计学习理论的 VC 维理论和结构风险最小原理基 础上的,根据有限
的样本信息在模型的复杂性(即对特定训练样本的学习精度, Accuracy)和学习能力(即无错误地识别任
意样本的能力)之间寻求最佳折衷, 以期获得最好的推广能力(或称泛化能力)。著名的 SVM[light]工具
经常被用作文本分类研究,它包含 25178 个数据集, 以研究哪篇文章是被某期刊收纳的。此次研究的数
据集即是从[该网项目](http://download.joachims.org/svm_light/examples/example1.tar.gz)中下
载而得。将下载下来的数据切分训练集 train.data 和测试集 test.data,其中训练集包 含 2000 行观
测,测试集包含 600 行观测.由于下载下来的数据即为特征向量的格式,因此每一行代表的是一篇文章,
“class”代表因变量“是否被接受”,“features”中包含成对的 fv,每一对 fv 中包含一个某 words 出现
在语料库中对应的行数,以及该 words 在该篇文章 中的特证权重。Words 语料库中包含 9947 个单词,
对于这 9947 个单词,如果在 谋篇文章中出现过,则必然会有相应的特征值,若没有出现则特征值为 0。
参考 Libsvm 项目,得到支持向量机的训练模型为 train_mapper.py、 train_reducer.py 以及
model_encoder.py。接下来可以直接对 train.data 进行训练得训练模型,


	cat train.data | ./train_mapper.py | sort | \
    ./train_reducer.py | ./model_encoder.py > \
    /home/cufe15/cheng_huimin/hadoop_work1/svm /model

模型 model 中包含的参数会返回两个结果:keys 和 score,其中 keys 为每 一个 word 对应的行数,而
score 是该 word 属于类别 1 的隶属度,分值越大,代 表属于该类的置信度越大。对测试集数据的每一
行中出现的 word,求这篇文章中所有语料库单词属于类别 1 的隶属度之和,若该和大于 0,则属于类别
1,若 该和小于 0,则属于类别 0。据此可以编写其 mapper 程序,如下所示:



	#! /usr/bin/env python
	# -*- coding: utf-8 -*-
	import collections, math, sys, json, urllib2, os

	def main():
		model = json.loads(urllib2.urlopen(os.environ['MODEL']).readline().strip())
		#读入上一步训练的 model
		W = collections.defaultdict(float) #建立一个新字典
		for f, w in model['parameters'].items(): #读入 model 中参数"parameters"的内容
			W[f] = w #提取 model 中参数"parameters"的内容,并存在一个新的字典 W 中
		for line in sys.stdin:
			x = json.loads(line)
			#计算 class 的预测值,若和大于 0,则预测值为 1,若和小于 0,则预测值为 0.
			prediction = 1 if 0. < sum([W[j] * x["features"][j] for j in x["features"].keys()]) else 0
			#最终输出 3 列,第一列为原始数据的"id",第二列为预测值,第三列为真实类别
			print '%s\t%d\t%d' % (x["id"], prediction, x["class"])

	if __name__ == '__main__':
		main()




运行该程序并将结果保存到 m1 中

	cat test.data | ./test_mapper.py >m1


其中,第一列表示每篇文章的“id”,第二列表示预测的 class 情况,第三 列表示真实的 class 情况,接
下来的工作就是如何利用这些信息计算出误判率。 分析思路如下:对于一篇文章而言,若其本身 class
为 1,而预测的 class 也为 1,那么此 时就为 True Positive,反之则为 False Positive;,若其本身
class 为 0,而 预测的 class 也为 0,那么此时就为 True Negative,反之则为 False Negative。

根据以上信息而已计算准确率(accuracy)、精确率(precision)、召回率 (recall)和 F1 值 4 个评价
分类性能的指标。准确率(accuracy)的定义是对于给定的测试数据集,分类器正确分类的样本 数与总样
本数之比,精确率(precision)计算的是所有"正确被分类的 item(TP)" 占所有"实际被分类到 1 的
(TP+FP)"的比例,召回率(recall)计算的是所有"正确 被分类的 item(TP)"占所有"应该被分类到 1 的
item(TP+FN)"的比例, F1 值就是 精确值和召回率的调和均值。接下来计算混淆矩阵以及 4 个指标的
程序如下所示:


	#! /usr/bin/env python

	# -*- coding: utf-8 -*-
	import math, sys, json, datetime, urllib2, os
	def main():
		matrix = { "TP": 0, "FP": 0, "TN": 0, "FN": 0 } #新建一个 keyvalue 型的字典
		for line in sys.stdin:
			trial = line.strip().split('\t') #按行读入数据
			prediction = int(trial[1]) #第二列为预测值
			klass = int(trial[2]) #第三列为真实值
			if klass == 1:
				if prediction == 1:
					matrix["TP"] += 1 #True Positive 加 1
				else:
					matrix["FN"] += 1 #False Positive 加 1
			else:
				if prediction == 0:
					matrix["TN"] += 1 #True Negative 加 1
				else:
					matrix["FP"] += 1 #False Negative 加 1
		true_positives = matrix["TP"]
		false_negatives = matrix["FN"]
		true_negatives = matrix["TN"]
		false_positives = matrix["FP"]
		sum_positives = true_positives + false_negatives
		sum_negatives = false_positives + true_negatives
		sum_trials = sum_positives + sum_negatives #求总样本数
		#求分类器正确分类的样本数与总样本数之比为准确率(accuracy)
		accuracy = float(true_positives + true_negatives) / float(sum_trials)
		#求所有"正确被分类的 item(TP)"占所有"实际被分类到 1 的(TP+FP)"的比例,即为精确率 precision
		precision = float(true_positives) / float(sum_positives)
		#求召回率(recall),即所有"正确被分类的 item(TP)"占所有"应该被分类到 1 的item(TP+FN)"的比例
		recall = float(true_positives) / float(true_positives + false_negatives)
		beta_squared = math.pow(1., 2.)
		p = precision
		r = recall
		f_measure = (1. + beta_squared) * ((p * r) / ((beta_squared * p) + r))
		#输出结果,不仅输出TP、FP、FN和TN,同时输出准确率accuracy、精确率precision、 召回率 recall 以及F值
		print "%d trials: TP=%d, FP=%d, FN=%d, TN=%d, acc=%3.2f, P=%3.2f, R=%3.2f,\
				F1 = %3.2f" % (sum_trials, true_positives, false_positives, false_negatives,\ 				true_negatives, accuracy, precision,recall, f_measure)

	if __name__ == '__main__':
		main()

运行程序如下所示:


	cat test.data | ./test_mapper.py | sort | ./test_reducer1.py

从输出结果中可得,混淆矩阵如下所示:

	     1     0
	1   295   15
	0    5    285
准确率 accuracy 为 97%,精确率 precision 为 95%,召回率 recall 为 98%, F1 值为 97%。
综上所述,支持向量机文本分类结果无论是根据准确率、精确率还是召回率 来进行评价,预测结果均高于 95%,准确度较高。
（感谢中央财经大学成慧敏提供素材和案例。）
