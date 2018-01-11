---
title: 论文阅读：Article De-duplication Using Distributed Representations
date: 2018-01-18
layout: post
permalink: /blog/2018/01/18/paper_read_dae_for_article_represention.html
categories: 论文阅读
tags: [推荐系统,embeding,dae]
excerpt: 雅虎日本新闻团队使用文章向量化表示用来做文章去重

---

###摘要说明
*雅虎日本新闻团队关于使用DAE(denoising auto-encoder)来做文章去重。背景为在新闻推荐中，多样性作为推荐系统一个重要评价指标，通常算法容易将相似的文章排在相近的地方(尤其是热点事件会有多个报道的情况)，使用户产生阅读疲劳(阅读信息下降，在线时间下降)，文章去重必不可少。向量化表示可以通过內积计算相似度，计算简单，利于线上部署*

###创新点
**dae通过类型有监督训练，效果变好**

###具体方案
输入均为bag of words vector。
原始的dae为：
![dae原理](http://ashan2012.github.io/images/paper_read_dae_for_article_represention/1.png)

论文中创新点在于加入文章的类别特征，同类别的相似度大于不同类别的，输入变为(x1,x2,x3)的pair,其中x1和x2为同类别的文章，x1和x3为不同类别的文章。整体的原理为:
![有监督的dae](http://ashan2012.github.io/images/paper_read_dae_for_article_represention/2.png)

公式中L损失函数为交叉熵,C为高斯噪声层.

###验证方式:
验证分为线下验证和线上验证。
线下验证（线下计算AUC)

对比方案为 文本相似性(Text-based cosines)（没提具体怎么做的，难道10万维向量相似度？). 运营标注了400个pair对，按照相似性分为5个级别.具体为:
![有监督的dae](http://ashan2012.github.io/images/paper_read_dae_for_article_represention/3.png)

可以看到，文本相似性只能判断出级别为4和级别和5的文章pair（全文文本）。按照二分类计算auc对比结果如下：
![有监督的dae](http://ashan2012.github.io/images/paper_read_dae_for_article_represention/4.png)

结果显示：在级别2和3上提升比较明显

线上a/b测试
测试指标:CTR (#clicks/#imps), depth (#imps/#sessions) and module CTR (#clicks/#sessions)
结果如下:
![有监督的dae](http://ashan2012.github.io/images/paper_read_dae_for_article_represention/5.png)

提升比较明显，depth明显增加，点击率增加，证明了推荐更加多样性，吸引了用户的兴趣




###参数设置
训练文本为400K文本(2015年3月)，词频排序前10万，去掉停用词。然后中间表示向量为 500维，输入向量为10万维，值为0,1.

###其他解决方案对比
###参考文献

