---

title: 基于svm的验证码识别—svm 
data: 2017-11-02
categories: python,SVM,Machine Learning
tags: [python,SVM,Machine Learning,captcha]

---

&emsp;&emsp;支持向量机，因其英文名为support vector machine，故一般简称SVM，通俗来讲，它是一种二类分类模型，其基本模型定义为特征空间上的间隔最大的线性分类器，其学习策略便是间隔最大化，最终可转化为一个凸二次规划问题的求解。

支持向量机理论部分函数，图片都主要参考  
[pluskid的SVM系列](http://blog.pluskid.org/?page_id=683)  
[解密SVM系列](http://blog.csdn.net/on2way/article/details/47729419)   
[简易解说拉格朗日对偶](https://www.cnblogs.com/90zeng/p/Lagrange_duality.html)  
[支持向量机推导过程](http://blog.sina.com.cn/s/blog_4298002e010144k8.html)  

使用到的概念：  
L2范数 
拉格朗日乘子法
KKT条件
强对偶
SMO算法
 


**简单了解SVM**  
&emsp;&emsp;给定一些数据点，它们分别属于两个不同的类，现在要找到一个线性分类器把这些数据分成两类。如果用x表示数据点，用y表示类别（y可以取1或者-1，分别代表两个不同的类），一个线性分类器的学习目标便是要在n维的数据空间中找到一个超平面（hyper plane），这个超平面的方程可以表示为
![](http://img.blog.csdn.net/20131107201211968)(w为法向量，b为常量)当f(x) 等于0的时候，x便是位于超平面上的点，而f(x)大于0的点对应 y=1 的数据点，f(x)小于0的点对应y=-1的点，  

(即线性方程:
<img src="https://i.loli.net/2017/11/17/5a0eacdf359a1.png" width = "300" height = "30" >)   

支持向量到超平面的距离：     
<img src="https://www.zhihu.com/equation?tex=%5Cfrac%7B1%7D%7B%7C%7Cw%7C%7C%7D%28w%5ETx_0%2Bb%29" width = "150" height = "40"  />
其中||w||为w的二阶范数（2阶范数简单理解即是欧式距离）  
（这样简单类推这个公式，在二维平面中，点到直线的距离 公式为![](https://ss1.baidu.com/6ONXsjip0QIZ8tyhnq/it/u=930595087,3467127615&fm=58) ）

现在已知支持向量到超平面的距离公式，我们要怎么样找到一个最优的超平面。
从直观上而言，这个超平面应该是最适合分开两类数据的直线。而判定“最适合”的标准就是这条直线离直线两边的数据的间隔最大。所以，得寻找有着最大间隔的超平面。

 如下图所示，中间的实线便是寻找到的最优超平面（Optimal Hyper Plane），其到两条虚线边界的距离相等，这个距离便是几何间隔r，两条虚线间隔边界之间的距离等于2r，而虚线间隔边界上的点则是支持向量。令函数间隔r等于1（之所以令等于1，是为了方便推导和优化，且这样做对目标函数的优化没有影响)由于这些支持向量刚好在虚线间隔边界上，所以它们满足
![](http://img.blog.csdn.net/20131111155244218)
而对于所有不是支持向量的点，则显然有
![](http://img.blog.csdn.net/20131111155205109)


![](http://img.blog.csdn.net/20140829141714944)

从而上述目标函数转化成了
![](http://img.my.csdn.net/uploads/201210/25/1351141837_7366.jpg)

上述目标函数等价于（w由分母变成分子，从而也有原来的max问题变为min问题，很明显，两者问题等价）：
![](http://img.my.csdn.net/uploads/201210/25/1351141994_1802.jpg) 

 因为现在的目标函数是二次的，约束条件是线性的，所以它是一个凸二次规划问题。这个问题可以用现成的QP (Quadratic Programming) 优化包进行求解。一言以蔽之：在一定的约束条件下，目标最优，损失最小。   

**深入SVM**
推导过程比较蛋疼，待研究
在求取有约束条件的优化问题时，拉格朗日乘子法（Lagrange Multiplier) 和KKT条件是非常重要的两个求取方法，对于等式约束的优化问题，可以应用拉格朗日乘子法去求取最优值；如果含有不等式约束，可以应用KKT条件去求取。当然，这两个方法求得的结果只是必要条件，只有当是凸函数的情况下，才能保证是充分必要条件。

一. 拉格朗日乘子法（Lagrange Multiplier) 和KKT条件

通常我们需要求解的最优化问题有如下几类：

(i) 无约束优化问题，可以写为:

    min f(x);  

(ii) 有等式约束的优化问题，可以写为:

     min f(x), 
     s.t. h_i(x) = 0; i =1, ..., n 

(iii) 有不等式约束的优化问题，可以写为：

      min f(x), 
      s.t. g_i(x) <= 0; i =1, ..., n
      h_j(x) = 0; j =1, ..., m

对于第(i)类的优化问题，使用求取f(x)的导数，然后令其为零，可以求得候选最优值，再在这些候选值中验证；如果是凸函数，可以保证是最优解。

对于第(ii)类的优化问题，常常使用的方法就是拉格朗日乘子法（Lagrange Multiplier) ，即把等式约束h_i(x)用一个系数与f(x)写为一个式子，称为拉格朗日函数，而系数称为拉格朗日乘子。通过拉格朗日函数对各个变量求导，令其为零，可以求得候选值集合，然后验证求得最优值。

对于第(iii)类的优化问题，常常使用的方法就是KKT条件。同样地，我们把所有的等式、不等式约束与f(x)写为一个式子，也叫拉格朗日函数，系数也称拉格朗日乘子，通过一些条件，可以求出最优值的必要条件，这个条件称为KKT条件。KKT条件是满足强对偶条件的优化问题的必要条件

那么KKT条件的定理是什么呢？就是如果一个优化问题在转变完后变成
![svm_pic3.PNG](https://i.loli.net/2017/11/21/5a13936dd579a.png)

其中g是不等式约束，h是等式约束（像上面那个只有不等式约束，也可能有等式约束）。那么KKT条件就是函数的最优值必定满足下面条件：

(1) L对各个x求导为零；
(2) h(x)=0;
(3) ∑αigi(x)=0，αi≥0
关于KKT条件的推导 [简易解说拉格朗日对偶（Lagrange duality）](https://www.cnblogs.com/90zeng/p/Lagrange_duality.html)

原问题优化为   
![](http://img.my.csdn.net/uploads/201210/25/1351142114_6643.jpg)
对w和b求偏导得到   
![](http://img.blog.csdn.net/20131107202220500)  
将偏导得到的代入原方程得到   
![](http://img.my.csdn.net/uploads/201210/25/1351142449_6864.jpg)

（2）求对α的极大，即是关于对偶问题的最优化问题。经过上面第一个步骤的求w和b，得到的拉格朗日函数式子已经没有了变量w，b，只有α。从上面的式子得到：
![](http://img.my.csdn.net/uploads/201206/02/1338605996_4659.jpg)

在求出a以后根据
![](http://img.my.csdn.net/uploads/201301/11/1357838666_9138.jpg)求出w
根据
![](http://img.my.csdn.net/uploads/201301/11/1357838696_3314.png)求出b

原超平面可表示为  
![svm_pic4.PNG](https://i.loli.net/2017/11/23/5a168c634b8f0.png)

**核函数**

![](http://blog.pluskid.org/wp-content/uploads/2010/09/two_circles.png)
前面我们介绍了线性情况下的支持向量机，它通过寻找一个线性的超平面来达到对数据进行分类的目的。不过，由于是线性方法，所以对非线性的数据就没有办法处理了。例如图中的两类数据，分别分布为两个圆圈的形状，不论是任何高级的分类器，只要它是线性的，就没法处理，SVM 也不行。因为这样的数据本身就是线性不可分的。   
如果用 X1 和 X2 来表示这个二维平面的两个坐标的话，我们知道一条二次曲线（圆圈是二次曲线的一种特殊情况）的方程可以写作这样的形式：
![svm_pic5.PNG](https://i.loli.net/2017/11/23/5a16906485992.png)

注意上面的形式，如果我们构造另外一个五维的空间，其中五个坐标的值分别为 Z1=X1, Z2=X21, Z3=X2, Z4=X22, Z5=X1X2，那么显然，上面的方程在新的坐标系下可以写作：
![svm_pic6.PNG](https://i.loli.net/2017/11/23/5a16913d93bb4.png)
关于新的坐标 Z ，这正是一个超平面的方程！  
将 X 按照上面的规则映射为 Z ，那么在新的空间中原来的数据将变成线性可分的，从而使用之前我们推导的线性分类算法就可以进行处理了。这正是 Kernel 方法处理非线性问题的基本思想。  
们通过一个映射 ϕ(⋅) 将其映射到一个高维空间中，数据变得线性可分了 
![svm_pic7.PNG](https://i.loli.net/2017/11/23/5a1691d23687e.png)

刚才的方法稍想一下就会发现有问题：在最初的例子里，我们对一个二维空间做映射，选择的新空间是原始空间的所有一阶和二阶的组合，得到了五个维度；如果原始空间是三维，那么我们会得到 19 维的新空间（验算一下？），这个数目是呈爆炸性增长的，这给 ϕ(⋅) 的计算带来了非常大的困难，而且如果遇到无穷维的情况，就根本无从计算了。所以就需要 Kernel 出马了

设两个向量 x1=(η1,η2)T 和 x2=(ξ1,ξ2)T ，而 ϕ(⋅) 即是到前面说的五维空间的映射，因此映射过后的内积为：
![svm_pic8.PNG](https://i.loli.net/2017/11/23/5a16939e162e9.png)   
同时
  ![svm_pic9.PNG](https://i.loli.net/2017/11/23/5a16943560a92.png)  

二者有很多相似的地方。区别在于什么地方呢？一个是映射到高维空间中，然后再根据内积的公式进行计算；而另一个则直接在原来的低维空间中进行计算，而不需要显式地写出映射后的结果。回忆刚才提到的映射的维度爆炸，在前一种方法已经无法计算的情况下，后一种方法却依旧能从容处理，甚至是无穷维度的情况也没有问题。

我们把这里的计算两个向量在映射过后的空间中的内积的函数叫做核函数 (Kernel Function) ，例如，在刚才的例子中，我们的核函数为：
![svm_pic10.PNG](https://i.loli.net/2017/11/23/5a169594b3714.png)

核函数能简化映射空间中的内积运算——刚好“碰巧”的是，在我们的 SVM 里需要计算的地方数据向量总是以内积的形式出现的。对比刚才我们写出来的式子，现在我们的分类函数为：

![svm_pic11.PNG](https://i.loli.net/2017/11/23/5a16987ceecfb.png)

其中 α 由如下 dual 问题计算而得：  
![svm_pic12.PNG](https://i.loli.net/2017/11/23/5a16997fd805c.png)  
这样一来计算的问题就算解决了，避开了直接在高维空间中进行计算，而结果却是等价的

通常人们会从一些常用的核函数中选择（根据问题和数据的不同，选择不同的参数，实际上就是得到了不同的核函数）
多项式核：
高斯核：
线性核：



**松弛变量**


**通过SMO算法求出α**


**验证码识别**  
验证码的处理跟之前KNN的处理一样，svm算法使用sklearn包里的svm算法

    # SVM Classifier using cross validation
    def svm_cross_validation(train_x, train_y):
    	from sklearn.model_selection import GridSearchCV
    	from sklearn.svm import SVC
    
    	model = SVC(kernel='rbf', probability=True,decision_function_shape='ovo')
    	param_grid = {'C': [1e-3, 1e-2, 1e-1, 1, 10, 100, 1000], 'gamma': [0.001, 0.0001]}
    	grid_search = GridSearchCV(model, param_grid, n_jobs=1, verbose=1)
    	grid_search.fit(train_x, train_y)
    	best_parameters = grid_search.best_estimator_.get_params() #获取最佳参数
    	for para, val in best_parameters.items():
    		print
    		para, val
    	model = SVC(kernel='rbf', C=best_parameters['C'], gamma=best_parameters['gamma'], probability=True,decision_function_shape='ovo')
    	model.fit(train_x, train_y)
   	 	return model

