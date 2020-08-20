---
layout:     post
title:      TensorFlow Error 1
subtitle:   "calling tf.nn.dynamic_rnn in the same scope twice"
date:       2020-08-20
author:     JH
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Machine Learning
    
---
```shell
tensorflow.python.framework.errors_impl.AlreadyExistsError: Resource __per_step_8/BiLSTM/bidirectional_rnn/fw/fw/while/fw/coupled_input_forget_gate_lstm_cell/ArithmeticOptimizer/AddOpsRewrite_Leaf_1_add_2/tmp_var/N10tensorflow19TemporaryVariableOp6TmpVarE
         [[{{node BiLSTM/bidirectional_rnn/fw/fw/while/fw/coupled_input_forget_gate_lstm_cell/ArithmeticOptimizer/AddOpsRewrite_Leaf_1_add_2/tmp_var}}]]
         [[{{node crf_loss/Mean}}]]
```
问题描述：My problem was also caused by the model size, disregard the "calling tf.nn.dynamic_rnn in the same scope twice" in my previous comment.

问题解决：
```python
config = tf.compat.v1.ConfigProto()
off = rewriter_config_pb2.RewriterConfig.OFF
config.graph_options.rewrite_options.memory_optimization = off
```