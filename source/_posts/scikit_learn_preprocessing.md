---

title: scikit_learn preprocessing
data: 2017-10-26
categories: Machine Learning,python
tags: [Machine Learning,python,scikit_learn]

---

特征数据预处理
参考自（[预处理数据的方法总结（使用sklearn-preprocessing）](http://blog.csdn.net/sinat_33761963/article/details/53433799) ）

1. 标准化：去均值，方差规模化

Standardization标准化:将特征数据的分布调整成标准正太分布，也叫高斯分布，也就是使得数据的均值维0，方差为1.

标准化的原因在于如果有些特征的方差过大，则会主导目标函数从而使参数估计器无法正确地去学习其他特征。

标准化的过程为两步：去均值的中心化（均值变为0）；方差的规模化（方差变为1）。

在sklearn.preprocessing中提供了一个scale的方法，可以实现以上功能。

    from sklearn import preprocessing
    import numpy as np
    x = np.array([[1., -1., 2.],
      [2., 0., 0.],
      [0., 1., -1.]])
	x_scale = preprocessing.scale(x)
	>>>x_scale
	>>>array([[ 0.        , -1.22474487,  1.33630621],
       [ 1.22474487,  0.        , -0.26726124],
       [-1.22474487,  1.22474487, -1.06904497]])
	>>>x_scale.mean(axis=0)
	>>>array([ 0.,  0.,  0.])
	>>>x_scale.std(axis=0)
	>>>array([ 1.,  1.,  1.])

用数学公式计算的话   
    
	#calculate mean
    x_mean = x.mean(axis=0)
    # calculate variance 
    x_std = x.std(axis=0)
    # standardize X
    x1 = (x-x_mean)/x_std

x1的计算结果与上述使用preprocessing.scale 计算的结果一样