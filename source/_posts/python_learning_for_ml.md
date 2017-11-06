
---

title: 数据科学的完整学习路径—Python版 
data: 2017-10-21
categories: Machine Learning,python
tags: [Machine Learning,python]

---

待消化的内容

转自([数据科学的完整学习路径—Python版](http://dataunion.org/9805.html?utm_source=tuicool))

从Python菜鸟到Python Kaggler的旅程（译注：Kaggle是一个数据建模和数据分析竞赛平台）

假如你想成为一个数据科学家，或者已经是数据科学家的你想扩展你的技能，那么你已经来对地方了。本文的目的就是给数据分析方面的Python新手提供一个完整的学习路径。该路径提供了你需要学习的利用Python进行数据分析的所有步骤的完整概述。如果你已经有一些相关的背景知识，或者你不需要路径中的所有内容，你可以随意调整你自己的学习路径，并且让大家知道你是如何调整的。

步骤0：热身

开始学习旅程之前，先回答第一个问题：为什么使用Python？或者，Python如何发挥作用？
观看DataRobot创始人Jeremy在PyCon Ukraine 2014上的30分钟演讲，来了解Python是多么的有用。

步骤1：设置你的机器环境

现在你已经决心要好好学习了，也是时候设置你的机器环境了。最简单的方法就是从Continuum.io上下载分发包Anaconda。Anaconda将你以后可能会用到的大部分的东西进行了打包。采用这个方法的主要缺点是，即使可能已经有了可用的底层库的更新，你仍然需要等待Continuum去更新Anaconda包。当然如果你是一个初学者，这应该没什么问题。

如果你在安装过程中遇到任何问题，你可以在这里找到不同操作系统下更详细的安装说明。

步骤2：学习Python语言的基础知识

你应该先去了解Python语言的基础知识、库和数据结构。Codecademy上的Python课程是你最好的选择之一。完成这个课程后，你就能轻松的利用Python写一些小脚本，同时也能理解Python中的类和对象。

具体学习内容：列表Lists，元组Tuples，字典Dictionaries，列表推导式，字典推导式。
任务：解决HackerRank上的一些Python教程题，这些题能让你更好的用Python脚本的方式去思考问题。
替代资源：如果你不喜欢交互编码这种学习方式，你也可以学习谷歌的Python课程。这个2天的课程系列不但包含前边提到的Python知识，还包含了一些后边将要讨论的东西。

步骤3：学习Python语言中的正则表达式

你会经常用到正则表达式来进行数据清理，尤其是当你处理文本数据的时候。学习正则表达式的最好方法是参加谷歌的Python课程，它会让你能更容易的使用正则表达式。

任务：做关于小孩名字的正则表达式练习。

如果你还需要更多的练习，你可以参与这个文本清理的教程。数据预处理中涉及到的各个处理步骤对你来说都会是不小的挑战。

步骤4：学习Python中的科学库—NumPy, SciPy, Matplotlib以及Pandas

从这步开始，学习旅程将要变得有趣了。下边是对各个库的简介，你可以进行一些常用的操作：

•根据[NumPy教程](http://scipy.github.io/old-wiki/pages/Tentative_NumPy_Tutorial)进行完整的练习，特别要练习数组arrays。这将会为下边的学习旅程打好基础。
•接下来学习[Scipy教程](https://docs.scipy.org/doc/scipy/reference/tutorial/)。看完Scipy介绍和基础知识后，你可以根据自己的需要学习剩余的内容。
•这里并不需要学习Matplotlib教程。对于我们这里的需求来说，Matplotlib的内容过于广泛。取而代之的是你可以学习[这个笔记](http://nbviewer.jupyter.org/github/jrjohansson/scientific-python-lectures/blob/master/Lecture-4-Matplotlib.ipynb)中前68行的内容。
•最后学习Pandas。Pandas为Python提供DataFrame功能（类似于R）。这也是你应该花更多的时间练习的地方。Pandas会成为所有中等规模数据分析的最有效的工具。作为开始，你可以先看一个关于Pandas的[10分钟简短介绍](http://pandas.pydata.org/pandas-docs/stable/10min.html)，然后学习一个更详细的[Pandas教程](http://www.gregreda.com/2013/10/26/intro-to-pandas-data-structures/)。
您还可以学习两篇博客[Exploratory Data Analysis with Pandas](http://www.analyticsvidhya.com/blog/2014/08/baby-steps-python-performing-exploratory-analysis-python/)和[Data munging with Pandas](https://www.analyticsvidhya.com/blog/2016/01/complete-tutorial-learn-data-science-python-scratch-2/)中的内容。

额外资源：
•如果你需要一本关于Pandas和Numpy的书，建议Wes McKinney写的“Python for Data Analysis”。
•在Pandas的文档中，也有很多Pandas教程，你可以在这里查看。

任务：尝试解决哈佛CS109课程的[这个任务](http://nbviewer.jupyter.org/github/cs109/2014/blob/master/homework/HW1.ipynb)。

步骤5：有用的数据可视化

参加CS109的[这个课程](http://cm.dce.harvard.edu/2015/01/14328/L03/screen_H264LargeTalkingHead-16x9.shtml)。你可以跳过前边的2分钟，但之后的内容都是干货。你可以根据[这个任务](http://nbviewer.jupyter.org/github/cs109/2014/blob/master/homework/HW2.ipynb)来完成课程的学习。

步骤6：学习Scikit-learn库和机器学习的内容

现在，我们要开始学习整个过程的实质部分了。Scikit-learn是机器学习领域最有用的Python库。这里是该库的简要概述。完成哈佛CS109课程的课程10到课程18，这些课程包含了机器学习的概述，同时介绍了像回归、决策树、整体模型等监督算法以及聚类等非监督算法。你可以根据各个课程的任务来完成相应的课程。

额外资源：

•如果说有那么一本书是你必读的，推荐Programming Collective Intelligence(集体智慧编程)。这本书虽然有点老，但依然是该领域最好的书之一。
•此外，你还可以参加来自Yaser Abu-Mostafa的机器学习课程，这是最好的机器学习课程之一。如果你需要更易懂的机器学习技术的解释，你可以选择来自Andrew Ng的机器学习课程，并且利用Python做相关的课程练习。
•Scikit-learn的教程

任务：尝试Kaggle上的这个[挑战](https://www.kaggle.com/c/data-science-london-scikit-learn)

步骤7：练习，练习，再练习

恭喜你，你已经完成了整个学习旅程。

你现在已经学会了你需要的所有技能。现在就是如何练习的问题了，还有比通过在Kaggle上和数据科学家们进行竞赛来练习更好的方式吗？深入一个当前Kaggle上正在进行的比赛，尝试使用你已经学过的所有知识来完成这个比赛。

步骤8：深度学习

现在你已经学习了大部分的机器学习技术，是时候关注一下深度学习了。很可能你已经知道什么是深度学习，但是如果你仍然需要一个简短的介绍，可以看[这里](http://dataunion.org/9805.html?utm_source=tuicool)。

我自己也是深度学习的新手，所以请有选择性的采纳下边的一些建议。deeplearning.net上有深度学习方面最全面的资源，在这里你会发现所有你想要的东西—讲座、数据集、挑战、教程等。你也可以尝试参加Geoff Hinton的课程，来了解神经网络的基本知识。

附言：如果你需要大数据方面的库，可以试试Pydoop和PyMongo。大数据学习路线不是本文的范畴，是因为它自身就是一个完整的主题。


