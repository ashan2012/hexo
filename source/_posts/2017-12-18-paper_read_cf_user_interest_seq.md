---
title: 论文阅读：Collaborative Filtering Recommendation on Users’ Interest Sequences
date: 2017-12-18
layout: post
permalink: /blog/2017/12/18/paper_read_cf_on_user_interest_seq.html
categories: 论文阅读
tags: [推荐系统,协同过滤,cf]
excerpt: 关于cf中用户兴趣变化的分析

---

*哈尔滨工程大学的几位同学的研究，看了开拓一下思路*

###摘要说明

论文主要是基于用户的行为序列，在cf的相似度中考虑用户行为模式的动态变化，通过提取两个用户相同的子模式（这里应该是最长公共子串），和相同子模式的个数等。

###具体方案



###参考文献

[1]**Tian Q, Zi-Ke Z, Guang C. Information Filtering via a Scaling-Based Function. PLoS ONE. 2013; 8(5)** 
A scaling-based algorithm with tunable parameters was introduced to promote personalized recommendation in solving the accuracy-diversity dilemma, presenting a high novelty and solving cold-start problem 

[2]**Koren Y. Collaborative Filtering with Temporal Dynamics. In: Proceedings of the 15th ACM SIGKDD International Conference on Knowledge Discovery and Data Mining. KDD’09. New York, NY, USA: ACM; 2009. p. 447–456. Available from: http://doi.acm.org/10.1145/1557019.1557072.**

动态时间信息的应用，在cf方法之中

[3]**Campos PG, Díez F, Cantador I. Time-aware Recommender Systems: A Comprehensive Survey and
Analysis of Existing Evaluation Protocols. User Modeling and User-Adapted Interaction. 2014 Feb; 24
(1–2):67–119. Available from: http://dx.doi.org/10.1007/s11257-012-9136-x. doi: 10.1007/s11257-012-
9136-x**  时间信息在cf的应用

[4] **Ding Y, Li X. Time Weight Collaborative Filtering. In: Proceedings of the 14th ACM International Conference on Information and Knowledge Management. CIKM’05. New York, NY, USA: ACM; 2005. p. 485–
492. Available from: http://doi.acm.org/10.1145/1099554.1099689.**  时间权重，越近的权重越大

[5]**Ding Y, Li X, Orlowska ME. Recency-based Collaborative Filtering. In: Proceedings of the 17th Australasian Database Conference—Volume 49. ADC’06. Darlinghurst, Australia, Australia: Australian Computer Society, Inc.; 2006. p. 99–107. Available from: http://dl.acm.org/citation.cfm?id=1151736.
1151747.**

时间权重

[6]**Cheng J, Liu Y, Zhang H, Wu X, Chen F. A New Recommendation Algorithm Based on User’s Dynamic
Information in Complex Social Network. Mathematical Problems in Engineering. 2015; 2015:6. doi: 10.
1155/2015/281629** 用户的动态信息利用
