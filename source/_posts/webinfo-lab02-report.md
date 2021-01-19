---
title: webinfo-lab02-report
date: 2021-01-03 22:59:41
tags: webinfo
---
webinfo lab2 实验报告
小组成员：PB17030889 吉志远

本实验要求以给定的英文文本数据集为基础，实现一个信息抽取系统。

<!--more-->

### 任务描述

信息抽取系统的主要功能是从文本中抽取出特定的事实信息，我们称之为实体。然而，在大多数的应用中，不但要识别文本中的实体，还要确定这些实体之间的关系，我们称其为实体关系抽取。在本次实验中，关系抽取为必做内容，实体识别为可选做内容。

本次实验仅完成了实体关系抽取部分，在测试系统上有56%的正确率

### 文件组


```bash
exp2:
│  .gitignore
│  README.md
│
├─.idea/
│
├─bert/
│  │  args.py
│  │  extract_feature.py
│  │  graph.py
│  │  modeling.py
│  │  optimization.py
│  │  tokenization.py
│  │  __init__.py
│  │
│  └─__pycache__/
│
├─data/
│      test.txt
│      train.txt
│
├─output/
│      relation.h5
│      tmp_graph11
│      吉志远-PB17030889-1.txt
│
├─src/
│   │  att.py
│   │  relation.py
│   │  requirements.txt
│   │
│   └─__pycache__/
│
│
└─multi_cased_L-12_H-768_A-12
```

### 文件说明
bert/
本模型采用文本分类深度学习模型，结合BERT预训练模型的[BERT实现](https://github.com/percent4/people_relation_extract)来进行关系抽取。其中bert模型数据为[google-research/bert](https://github.com/google-research/bert/blob/master/multilingual.md)

模型结构：BERT + 双向GRU + Attention + FC

1. `bert/args.py` 读取bert模型multi_cased_L-12_H-768_A-12和配置文件bert_config.json
2. `bert/extract_feature.py`根据数据生成bert-vector用于训练
3. `bert/graph.py` 卷积网络，生成tmp_graph文件
4. `bert/modeling.py` google bert model API
5. `bert/optimization.py`google bert optimization(weight updates)
6. `bert/tokenization.py` google bert tokenization class
7. `att.py` 利用Keras构造注意力机制层类`Attention`
8. `relations.py` 定义关系字典`relation_dict`，负责数据的输入和对文档进行适当的预处理，以可以被BERT模型读取和训练

### 函数说明

读取文件

```python
# 读取训练数据，分词和预处理，找到训练数据的关系
def get_train_set():
    f = open("./train.txt", "r", encoding='utf-8')
    texts = [] # 文本
    rel = [] # 关系标签
    line = f.readline()
    while line:
        index = line.find(' ')
        line = line[index+2:-2]
        texts.append(line)
        line = f.readline()
        line = line.split('(')[0]
        rel.append(relation_dict[line])
        line = f.readline()
    f.close()
    train_df = pd.DataFrame({'label':rel[0:5120], 'text':texts[0:5120]})
    test_df = pd.DataFrame({'label':rel[5120:], 'text':texts[5120:]})
    
    return train_df, test_df
```
训练模型

```python
# 训练数据
def train_model():
    train_df, test_df = get_train_set()
    # 实例化BertVector类，开始构建模型
    bert_model = BertVector(pooling_strategy="NONE", max_seq_len=80)
    print('begin encoding')
    f = lambda text: bert_model.encode([text])["encodes"][0]

    train_df['x'] = train_df['text'].apply(f)
    test_df['x'] = test_df['text'].apply(f)
    print('end encoding')

    x_train = np.array([vec for vec in train_df['x']])
    x_test = np.array([vec for vec in test_df['x']])
    y_train = np.array([vec for vec in train_df['label']])
    y_test = np.array([vec for vec in test_df['label']])

    num_classes = 10
    y_train = to_categorical(y_train, num_classes)
    y_test = to_categorical(y_test, num_classes)

    # 模型结构：BERT + 双向GRU + Attention + FC
    inputs = Input(shape=(80, 768, ))
    gru = Bidirectional(GRU(128, dropout=0.2, return_sequences=True))(inputs)
    attention = Attention(32)(gru)
    output = Dense(10, activation='softmax')(attention)
    model = Model(inputs, output)

    # 模型可视化
    # from keras.utils import plot_model
    # plot_model(model, to_file='model.png')

    model.compile(loss='categorical_crossentropy', optimizer=Adam(), metrics=['accuracy'])

    # 模型训练以及评估
    model.fit(x_train, y_train, batch_size=8, epochs=30)
    model.save('relation.h5')
    print(model.evaluate(x_test, y_test))
```

预测
```python
def predict():
    # 读取预测文本
    texts = []
    f = open('test.txt', 'r', encoding='utf-8')
    line = f.readline()
    while line:
        index = line.find(' ')
        line = line[index+2:-2]
        texts.append(line)
        line = f.readline()
    f.close()
        
    # 加载模型
    model = load_model('relation.h5', custom_objects={"Attention":Attention})

    # 利用BERT提取句子特征
    bert_model = BertVector(pooling_strategy="NONE", max_seq_len=80)
    predict_df = pd.DataFrame({'text':texts})
    f = lambda text: bert_model.encode([text])["encodes"][0]
    predict_df['x'] = predict_df['text'].apply(f)
    x_predict = np.array([vec for vec in predict_df['x']])

    # 模型预测并输出预测结果
    rel_dict = {v:k for k,v in relation_dict.items()}
    predicted = model.predict(x_predict)
    y_predicted = []
    for y in predicted:
        y_predicted.append(rel_dict[np.argmax(y)])
    
    # 写入结果
    with open('output.txt', 'w', encoding='utf-8') as f:
        for rel_predicted in y_predicted:
            f.write(rel_predicted + '\n')
```

### 模型算法

#### 模型

本项目所采用的模型为：BERT + 双向GRU + Attention + FC，其中BERT的全称是Bidirectional Encoder Representation from Transformers，即双向Transformer的Encoder，用来提取文本的特征，Attention为注意力机制层，FC为全链接层，模型的结构图为

![模型结构示例图](![620cd20bbc814fc586c54233dda860cc-1.jpg](https://i.loli.net/2021/01/04/qMO4oKwgtsBeLD5.jpg)

bert文件夹下的源码文件可以负责tokenizer，padding, masking以及BERT模型产生输出向量，在模型训练中，原始数据作为GRU层的输入，采用Masked LM和Next Sentence Prediction两种方法分别捕捉词语和句子级别的表征，用于关系提取，具体运行步骤

1. 使用Masked LM方式将语料中的某一部分的词语掩盖住，模型通过上下文预测被掩盖的词语，从而训练出初步的模型。
2. 在语料中选出连续的上下文语句，并使用Transformer模块识别语句的连续性。
3. 通过（1）和（2）实现通过上下文进行双向预测的预训练语言表征模型。
4. 然后通过少量经过标记的数据以监督学习的方式对模型进行fine-tuning

GRU层：将Input_Layer层句子中的每一个字符输入作为character embedding。这样的模型对每一个句子输入做训练，加入字级别的Attention

Attention层：生成一个权重向量，通过与这个权重向量相乘，使每一次迭代中的词汇级的特征合并为句子级的特征

Dense层：全链接层

#### 优化

这里采用了google-bert的[优化文件](https://github.com/google-research/bert/blob/master/optimization.py)，实验成员并没有也不敢做什么优化

BERT模型采用了权重衰减的Adam优化器（[AdamW](https://arxiv.org/abs/1711.05101)），它采用学习率计划，即从0开始设定权重并逐步下降到0

### 运行测试和结果

#### 运行环境

tensorflow的1.x和2.0之后的版本有一定区别，由于python3.8及以上的版本仅支持tensorflow2.0以上的版本，本实验用到的bert模型为tensorflow1.13。所以在conda activate py37的环境下运行

```bash
# requirements.txt 
tensorflow==1.13.1
keras==2.3.1
pandas==0.23.4
matplotlib==2.2.4
numpy==1.16.2
scikit-learn
xlrd
```

#### 运行截图
![sm.ms/image/lgQCZsPIz1LH3Y2](https://i.loli.net/2021/01/04/lgQCZsPIz1LH3Y2.png)

预编码构建模型

![___QNQBJIJ_M6VE``_93DTG.png](https://i.loli.net/2021/01/04/zwcJoZeKEStOAdr.png)

使用训练集的全部数据对模型进行一次完整训练，共训练30次
![image-20210104104833003.png](https://i.loli.net/2021/01/04/lNSmdGyJwoZx2VY.png)