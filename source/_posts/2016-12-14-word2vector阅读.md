---
title: 源码阅读系列之word2vector
date: 2016-12-14
layout: post
permalink: /blog/2016/12/14/word2vector阅读.html
categories: 源码阅读
tags: [word2vector, wordtovector, 词向量, 源码, 源代码, 负抽样, cbow ]
excerpt: 根据word2vector的源码介绍其原理以及如何应用

---

##基本的参数介绍

+ **size** 输出的vector的维度
+ **train** 训练的文本，已经分好词，按照所有的行分割，一行最多10000个词
+ **save-vocab** 将文本中读取的词出现次数保存
+ **read-vacab** 不从训练的文本读取词次数，直接从文件中读取词的次数
+ **debug** 输出更多信息
+ **binary** 以二进制的方式保存模型，其实就是每个词语对应的vector
+ **cbow**是否使用cbow模型
+ **alpha** 梯度下降的初始学习率，默认的skip-gram 0.025,cbow 0.05,实际是每处理10000个词，   alpha慢慢减小
+ **window** 前后词的窗口最大长度(context(w))的长度，实际会有一个0-window的随机
+ **sample** 采样的时候高频词会有一定的丢弃概率
+ **hs** 是否使用Hierarchical Softmax，默认等于0，不使用
+ **negtive** 负采样词语个数
+ **threads** 线程个数，默认12
+ **iter** 迭代次数，默认是5
+ **min-count** 低于min-count的词语直接丢弃
+ **classes** 输出的是word的类别，不是vector

##示例
**./word2vec -train data.txt -output vec.txt -size 200 -window 5 -sample 1e-4 -negative 5 -hs 0 -binary 0 -cbow 1 -iter 3**

1. tricks 提前算好sigmod函数:
   
    	
```cpp
    expTable = (real *)malloc((EXP_TABLE_SIZE + 1) * sizeof(real));
    for (i = 0; i < EXP_TABLE_SIZE; i++) {
        expTable[i] = exp((i / (real)EXP_TABLE_SIZE * 2 - 1) * MAX_EXP);
        expTable[i] = expTable[i] / (expTable[i] + 1); 
    }
```

   
##数据结构介绍
```cpp
   //softmax存储词语的huffuman编码信息
	struct vocab_word 
	{  
  		long long cn; //词频  
  		int *point; //huffman编码对应内节点的路径  
  		char *word, *code, codelen;//huffman编码  
	};
```
数组vocab存取词语信息，每次1000的量增加

##函数介绍
  ```
  TrainModel()
  ```

读取每个词语，存入vacab，和vacab_hash中.

从词表中读入信息:
读入代码:

```cpp

    //我觉得此处有问题，为什么没有读入词频
	vocab[vocab_size].word = (char *)calloc(length, sizeof(char))
	strcpy(vocab[vocab_size].word, word); 
	vocab[vocab_size].cn = 0;
	vocab_size++; 
```

hash采用线性冲突法。顺序往下找:

```cpp

	hash = GetWordHash(word); 
	while (vocab_hash[hash] != -1) hash = (hash + 1) % vocab_hash_size;
	vocab_hash[hash] = vocab_size - 1;
```

其中hash值的获得很巧妙:

```cpp

	int GetWordHash(char *word) {  
		unsigned long long a, hash = 0;
		for (a = 0; a < strlen(word); a++) hash = hash * 257 + word[a];
		hash = hash % vocab_hash_size;
		return hash;
	}
```

读取词语结束后，``SortVocab``所有的词语排序，然后去掉出现次数小于min_count的词语，然后重新调整vacab_hash,同时开辟存储二叉树的节点，路径的排序

从训练级中读取数据:
相同的思路但是需要在词语数目大的时候，删减词语集合

```cpp

	if (vocab_size > vocab_hash_size * 0.7) ReduceVocab();
    //默认min_reduce=1
    void ReduceVocab() {
  		int a, b = 0;
  		unsigned int hash;
  		for (a = 0; a < vocab_size; a++) if (vocab[a].cn > min_reduce) {
    		vocab[b].cn = vocab[a].cn;
    		vocab[b].word = vocab[a].word;
   			 b++;
  		} else free(vocab[a].word);
  		vocab_size = b;
 		for (a = 0; a < vocab_hash_size; a++) vocab_hash[a] = -1; 
  		for (a = 0; a < vocab_size; a++) {
    	// Hash will be re-computed, as it is not actual
   			hash = GetWordHash(vocab[a].word);
   		 	while (vocab_hash[hash] != -1) hash = (hash + 1) % vocab_hash_size;
    	 	vocab_hash[hash] = a;
  	}
  	fflush(stdout);
 	 min_reduce++;
	}
```

huffuman树的建树很有意思:

```cpp

    //实际写一个6,5,4,3,2,1的过程就明白了
	void CreateBinaryTree() {
        long long a, b, i, min1i, min2i, pos1, pos2, point[MAX_CODE_LENGTH];
        char code[MAX_CODE_LENGTH];
        long long *count = (long long *)calloc(vocab_size * 2 + 1, sizeof(long long));
        long long *binary = (long long *)calloc(vocab_size * 2 + 1, sizeof(long long));
        long long *parent_node = (long long *)calloc(vocab_size * 2 + 1, sizeof(long long));
        for (a = 0; a < vocab_size; a++) count[a] = vocab[a].cn;
        for (a = vocab_size; a < vocab_size * 2; a++) count[a] = 1e15;
        pos1 = vocab_size - 1;
        pos2 = vocab_size;
        // Following algorithm constructs the Huffman tree by adding one node at a time
        for (a = 0; a < vocab_size - 1; a++) {
            // First, find two smallest nodes 'min1, min2'
            if (pos1 >= 0) {
                if (count[pos1] < count[pos2]) {
                    min1i = pos1;
                    pos1--;
                } else {
                    min1i = pos2;
                    pos2++;
                }
           } else {
               min1i = pos2;
               pos2++;
           }   
           if (pos1 >= 0) {
               if (count[pos1] < count[pos2]) {
                   min2i = pos1;
                   pos1--;
              } else {
                  min2i = pos2;
                  pos2++;
              }
           } else {
               min2i = pos2;
               pos2++;
           }      
          count[vocab_size + a] = count[min1i] + count[min2i]; //min1,min2为每次组建树        
                                                               //的叶子节点，其中min2i    
                                                               //为较大的那一个叶子节点
          parent_node[min1i] = vocab_size + a;  //存储父子关系
          parent_node[min2i] = vocab_size + a;
          binary[min2i] = 1;   //存储0，1作为路径
       }
       // Now assign binary code to each vocabulary word
       //point太牛逼,point第一位一直为vocab_size-2,为树的根节点，在code中不存储根节点
       
       for (a = 0; a < vocab_size; a++) {
         b = a;
         i = 0;
         while (1) {
            code[i] = binary[b];
            point[i] = b;
            i++;
            b = parent_node[b];
            if (b == vocab_size * 2 - 2) break;
         }
         vocab[a].codelen = i;
         vocab[a].point[0] = vocab_size - 2;
         for (b = 0; b < i; b++) {
             vocab[a].code[i - b - 1] = code[b];   //顺序倒过来，从根节点下一层往下到叶子      
                                                   //节点
             vocab[a].point[i - b] = point[b] - vocab_size; //meikandong
         }
      }
      free(count);
      free(binary);
      free(parent_node);
    }
```

接下来是负抽样的初始化,根据词的个数生成概率数组

```cpp

	void InitUnigramTable() {
  		int a, i;
  		long long train_words_pow = 0;
  		real d1, power = 0.75;
 		table = (int *)malloc(table_size * sizeof(int));
  		for (a = 0; a < vocab_size; a++) train_words_pow += pow(vocab[a].cn, power);
  		i = 0;
  		d1 = pow(vocab[i].cn, power) / (real)train_words_pow;
  		for (a = 0; a < table_size; a++) {
    		table[a] = i;
    		if (a / (real)table_size > d1) {
     			 i++;
     			 d1 += pow(vocab[i].cn, power) / (real)train_words_pow;
    		}
    	   if (i >= vocab_size) i = vocab_size - 1;
  		}
	}
    //生成table,[0,0,0,0,1,1,1,2]
```

训练
l1 ns中表示word在sys1neg中的起始位置，主要用来获取负样本的vector
l2 cbow表示syn1中的非叶子节点的位置,主要用来获取非叶子节点的vector
taget ns中当前词语
label ns中当前词语的label,
neu1  隐藏层的vector
nuu1e 误差累积项

两个模型cbow和skip-
具体的推理过程可以看[csdn 基于hs的模型](http://blog.csdn.net/itplus/article/details/37969979 "csdn 基于hs的模型")
其中cbow的模型结构如如下文:

![cbow结构图](http://ashan2012.github.io/images/word2vector/cbow.png)

cbow更新的伪代码如下:

![cbow的更新伪码](http://ashan2012.github.io/images/word2vector/cbowliucheng.png)
 
看一下使用hs的cbow模型:

```cpp  
   
	if (cbow) {  //train the cbow architecture
      // in -> hidden
      cw = 0;
      for (a = b; a < window * 2 + 1 - b; a++) if (a != window) {   //context(w)输入上下文相加得到隐藏层
        c = sentence_position - window + a;
        if (c < 0) continue;
        if (c >= sentence_length) continue;
        last_word = sen[c];
        if (last_word == -1) continue;
        for (c = 0; c < layer1_size; c++) neu1[c] += syn0[c + last_word * layer1_size];
        cw++;
      }   
      if (cw) {
        for (c = 0; c < layer1_size; c++) neu1[c] /= cw;   //隐藏层的变量除以输入的变量个数
        if (hs) for (d = 0; d < vocab[word].codelen; d++) {
          f = 0;
          l2 = vocab[word].point[d] * layer1_size;  //非叶子节点的位置
          // Propagate hidden -> output
          for (c = 0; c < layer1_size; c++) f += neu1[c] * syn1[c + l2];   //逻辑函数的因变量,Xw*参数,即非叶子节点向量
          if (f <= -MAX_EXP) continue;
          else if (f >= MAX_EXP) continue;
          else f = expTable[(int)((f + MAX_EXP) * (EXP_TABLE_SIZE / MAX_EXP / 2))];  //分类的概率值
          // 'g' is the gradient multiplied by the learning rate
          g = (1 - vocab[word].code[d] - f) * alpha;    
          // Propagate errors output -> hidden
          for (c = 0; c < layer1_size; c++) neu1e[c] += g * syn1[c + l2];  //误差累积项，用来更新词向量，上下文词的向量
          // Learn weights hidden -> output
          for (c = 0; c < layer1_size; c++) syn1[c + l2] += g * neu1[c];   //更新非叶子节点的向量，即参数向量
        }   
       //文中此处为负抽样的代码
       //更新所有上下文词的词向量
        // hidden -> in
        for (a = b; a < window * 2 + 1 - b; a++) if (a != window) {
          c = sentence_position - window + a;
          if (c < 0) continue;
          if (c >= sentence_length) continue;
          last_word = sen[c];
          if (last_word == -1) continue;
          for (c = 0; c < layer1_size; c++) syn0[c + last_word * layer1_size] += neu1e[c]; //这一步更新
        }
```
 

负抽样的代码如下:

```cpp
          // NEGATIVE SAMPLING
        if (negative > 0) for (d = 0; d < negative + 1; d++) {
          if (d == 0) {
            target = word;
            label = 1;
          } else {
            next_random = next_random * (unsigned long long)25214903917 + 11; 
            target = table[(next_random >> 16) % table_size];
            if (target == 0) target = next_random % (vocab_size - 1) + 1;
            if (target == word) continue;
            label = 0;
          }   
          l2 = target * layer1_size;
          f = 0;
          for (c = 0; c < layer1_size; c++) f += neu1[c] * syn1neg[c + l2];
          if (f > MAX_EXP) g = (label - 1) * alpha;
          else if (f < -MAX_EXP) g = (label - 0) * alpha;
          else g = (label - expTable[(int)((f + MAX_EXP) * (EXP_TABLE_SIZE / MAX_EXP / 2))]) * alpha;
          for (c = 0; c < layer1_size; c++) neu1e[c] += g * syn1neg[c + l2];
          for (c = 0; c < layer1_size; c++) syn1neg[c + l2] += g * neu1[c];
        } 
```
   


附录，推导过程：
负抽样方法模型推导见:
[word2vector中的负抽样](http://blog.csdn.net/itplus/article/details/37998797)
大致思路为条件为上下文，正确的词语概率大，随机负抽样的概率小

![](http://ashan2012.github.io/images/word2vector/negtive1.png)

![](http://ashan2012.github.io/images/word2vector/negtive2.png)

![](http://ashan2012.github.io/images/word2vector/negtive3.png)

![](http://ashan2012.github.io/images/word2vector/negtive4.png)

cbow的推导过程:

![](http://ashan2012.github.io/images/word2vector/cbow_1.png)

![](http://ashan2012.github.io/images/word2vector/cbow_1.png)

![](http://ashan2012.github.io/images/word2vector/cbow_1.png)

![](http://ashan2012.github.io/images/word2vector/cbow_1.png)


写在结尾，以上推导过程来自
[csdn 皮果提博客](http://my.csdn.net/peghoty "csdn 皮果提博客")

结尾十问：***每次写完一个模型或者一个算法都要有问题***


1. ***wordtovecotor训练的词向量区别于one-hot模型的地方有哪些？***
   
    语义特性。wordtovecotr训练的词向量表示的是语义空间的点，点之间的远近关系通过词之间的共现关系训练得到。

1. ***训练方法上，为什么负抽样可以得到和window方法相似的效果？***
   
    训练空间上有语义传播性，所以在区分的不同语义的内容时，只需要区分一个就好。例如a和b关系较远，b和c关系较远，则a和c的关系也较远，这个负抽样也能达到效果的原因

1. ***wordtovector中如何霍夫曼树是为什么可以实现等同softmax的作用？***

    以三层霍夫曼树为例，每次拆分概率为p,q,r.最后叶子节点的概率和为: p.q+p(1-q)+(1-p)r+(1-p)(1-r)=1

1. ***wordtovector中词向量为什么可以表示语义空间的点？***

    霍夫曼树种每个节点的分裂表示的语义的一次分类，则经过整棵树的分类之后就到了叶子节点就是语义空间的一个点

1. ***wordtovector中对高频词和低频词处理的原则，以及为什么这么做?***
   
   低频词按照阈值丢掉，如果内容加载不下，提高阈值。
   高频词按照一定的概率丢弃。
   低频词本来就无法训练出来语义特征
   高频词按照论文的解释，高频词由于共现的词很多，语义特征不具有区分性。比如训练巴黎的时候，更希望和纽约一块训练，而不是和“的”一块儿训练
