---
layout:     post
title:      安装TensorFlow
subtitle:   在Linux上安装TensorFlow(如果不踩坑的话🙈🙊🙉)
date:       2020-08-20
author:     JH
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Machine Learning    
---

## TensorFlow 库的安装

### 1. cuda 10.0 安装
#### · [Nvidia官网 cuda](https://developer.nvidia.com/cuda-toolkit-archive) 自行下载cuda 10.0版本
### 2. cudnn 7.4.2安装 
#### · [Nvidia官网 cudnn](https://developer.nvidia.com/rdp/cudnn-archive) 自行下载cudnn 7.4.2版本
### 3. tensorflow 1.13.1安装 
#### · pip工具下载tensorflow-gpu版本，下载命令如下所示：
```python
pip install tensorflow-gpu==1.13.1
```
### 4. tensorflow测试
#### · tensorflow多GPU运行代码测试
```python
import tensorflow as tf
tf.__version__  # 查看tensorflow版本

with tf.device('/cpu:0'):
    a = tf.constant([1.0,2.0,3.0],shape=[3],name='a')
    b = tf.constant([1.0,2.0,3.0],shape=[3],name='b')

with tf.device('/gpu:1'):
    c = a+b
   
# 注意：allow_soft_placement=True表明：计算设备可自行选择，如果没有这个参数，会报错。

# 因为不是所有的操作都可以被放在GPU上，如果强行将无法放在GPU上的操作指定到GPU上，将会报错。

sess = tf.Session(config=tf.ConfigProto(allow_soft_placement=True,log_device_placement=True))

#sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))

sess.run(tf.global_variables_initializer())

print(sess.run(c))
```