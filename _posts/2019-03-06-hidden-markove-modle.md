---
layout: post
title:  "隐马尔可夫模型"
date:   2019-03-06 15:49:54
catalog:  true
tags:
    - 隐马尔可夫
    - HMM
    
---


# 一. 模型概要

机器学习最重要的的任务, 是根据一些已观察到的证据(如训练样本)来对感兴趣的未知变量进行估计和推测;  


 隐马尔可夫它假设某一时刻状态转移的概率只依赖于它的前一个状态, 隐马尔可夫模型的图结构如下:  

![](/images/ASR/hiddenMarkove/hmm.png)

 隐马尔可夫模型中的变量可分为两组:

- 状态变量(隐藏变量)

状态变量$\{x(t-1), x(t), x(t+1)...x(t+n)\}$, 其中 $x(t) \in \chi$ 表示第`t`时刻的系统状态. 通常假定状态变量是隐藏的, 不可观测的;  

- 观测变量

观测变量$\{y(t-1), y(t), y(t+1)...y(t+n)\}$, 其中$y(t) \in \gamma$表示第`t`时刻的观测值;

图中的箭头表示了变量间的依赖关系, 在任一时刻, 观测变量的取值仅依赖于状态变量, 即$y(t)$由 $x(t)$确定, 与其他状态变量及观测变量的取值无关;   
同时`t`时刻的状态$x(t)$仅依赖于`t-1`时刻的状态$x(t-1)$, 与`t-2`时刻状态无关, 即系统下一时刻的状态只由当前状态决定, 不依赖于以往的任何状态;  
所有变量的概率分布为:  

$$
P(x_t,y_t,...,x_{t+n},y_{t+n}) = 
P(x_t)P(y_t|x_t)
\prod_{i = 2}^n
P(x_i|x_{i-1})
P(y_i|x_i)
$$


初了结构信息, 欲确定一个隐马尔可夫模型还需要以下三组参数:  

- 状态转移概率  

> 模型在各个状态之间转换的概率

- 输出观测概率  

> 模型根据当前状态获取到的各个观测值概率  

- 初始状态概率  

> 模型在初始状态的各个状态的概率  

# 二. 实例

## 2.1 实例
假设你有一个住得很远的朋友，他每天跟你打电话告诉你他那天做了什么。你的朋友仅仅对三种活动感兴趣：公园散步，购物以及清理房间。他选择做什么事情只凭天气。你对于他所住的地方的天气情况并不了解，但是你知道总的趋势。在他告诉你每天所做的事情基础上，你想要猜测他所在地的天气情况。

你认为天气的运行就像一个马尔可夫链.其有两个状态 "雨"和"晴",但是你无法直接观察它们,也就是说,它们对于你是隐藏的。每天，你的朋友有一定的概率进行下列活动:"散步"、"购物"、"清理"。因为你朋友告诉你他的活动，所以这些活动就是你的观测数据。这整个系统就是一个隐马尔可夫模型HMM。

你知道这个地区的总的天气趋势,并且平时知道你朋友会做的事情.也就是说这个隐马尔可夫模型的参数是已知的.你可以用程序语言(Python)写下来:

```
states = ('Rainy', 'Sunny')

observations = ('walk', 'shop', 'clean')

start_probability = {'Rainy': 0.6, 'Sunny': 0.4}

transition_probability = {
    'Rainy' : {'Rainy': 0.7, 'Sunny': 0.3},
    'Sunny' : {'Rainy': 0.4, 'Sunny': 0.6},
}

observation_probability = {
    'Rainy' : {'walk': 0.1, 'shop': 0.4, 'clean': 0.5},
    'Sunny' : {'walk': 0.6, 'shop': 0.3, 'clean': 0.1},
}
```


## 2.2 计算方式
- 第一天的活动是散步, 而且第一天没有状态转移概率, 所以计算概率的方式以下雨为例: `start_probability['Rainy']*emission_probability['Rainy'][walk]`
这样第一天分别算出第一天晴天和下雨的概率分别为: 0.24和0.01

- 第二天的计算总结公式: 概率   = $x_t$状态输出序列概率 * 状态$x_t$向$x_{t+1}$状态转移概率 *  当前状态($x_{t+1}$)对应的观测概率


这里解释下状态输出序列概率和状态对应的观测概率的区别, 这是笔者自己定义的;

状态输出序列概率是指最后可观测现象到的总概率(状态序列和观测概率的结合), 比如第一天晴天的概率为0.24  

状态对应的观测概率是指一定状态下发生的观测现象的概率, 比如第一天在晴天的情况下, 对应的散步的概率为0.6  


## 2.3 python实现

```
# -*- coding:utf-8 -*-
# Filename: hmm.py
# Author：Sourcelink
# Date: 2019-03-05 15:24
 
states = ('Rainy', 'Sunny')
 
observations = ('walk', 'shop', 'clean')
 
start_probability = {'Rainy': 0.6, 'Sunny': 0.4}
 
transition_probability = {
    'Rainy' : {'Rainy': 0.7, 'Sunny': 0.3},
    'Sunny' : {'Rainy': 0.4, 'Sunny': 0.6},
    }
 
observation_probability = {
    'Rainy' : {'walk': 0.1, 'shop': 0.4, 'clean': 0.5},
    'Sunny' : {'walk': 0.6, 'shop': 0.3, 'clean': 0.1},
}


def viterbi(obs, states, start_p, trans_p, obs_p):
    """
    obs:     观测序列
    states:  隐状态
    start_p: 初始化隐状态概率
    trans_p: 隐状态转移概率
    obs_p:   状态对应观测概率
    """
    # 状态: 概率
    V = [{}]

    # 记录路径
    path = {}

    # 计算第一天的最大概率 t = 0时刻
    for x in states:
        V[0][x] = start_p[x] * obs_p[x][obs[0]]
        path[x] = [x]

    for t in range(1, len(obs)):
        V.append({})

        for x in states:
            # 假设x的状态是下雨, 结合x0状态的情况求x状态是下雨还是晴天的概率大
            # 概率  x0状态概率 = x0状态输出序列概率 * 状态x0向x状态转移概率 *  当前状态对应的观测概率
            (prob, state) = max([(V[t-1][x0] * trans_p[x0][x] * obs_p[x][obs[t]], x0) for x0 in states])
            V[t][x] = prob
            # print(V)
            # 将当前状态和上次状态保存, 并且以当前状态x为key保存
            path[x] = path[state] + [x]
            # print (path)

    
    (prob, state) = max([(V[len(obs) - 1][x], x) for x in states])
    return (prob, path[state])



if __name__ == '__main__':
    print(viterbi(observations, 
        states, start_probability, 
        transition_probability, 
        observation_probability))
```

```
(0.01344, ['Sunny', 'Rainy', 'Rainy'])
```

以上是笔者最后算出的结果, 有的朋友算出第二天晴天的概率比下雨的概率大, 觉得最后的输出序列应该是`['Sunny', 'Sunny', 'Rainy']`;  
但是那样是局部最优计算, 马尔可夫链的输出结果应该是以最后一天的输出概率大小来确定;  


该实例参考wiki[隐马尔可夫实例](https://zh.wikipedia.org/wiki/隐马尔可夫模型)  































