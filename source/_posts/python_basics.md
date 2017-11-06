---

title: python基础—Python 
data: 2017-10-25
categories: python
tags: [python]

---

python 30分钟入门教程：[Learn X in Y minutes](https://learnxinyminutes.com/docs/python/)

一、python解释器  
CPython：官方版本解释器，使用最广  
IPython：基于CPython之上的一个交互式解释器  
PyPy：目标是执行速度，对Python代码进行动态编译（注意不是解释）  
Jython：运行在Java平台上的Python解释器  
IronPython：运行在微软.Net平台上的Python解释器  

二、输入输出  
raw_input和print是在命令行下面最基本的输入和输出

三、数据类型和变量	  
1.整数：  
2.浮点数：  
3.字符串：转义字符\ & Python还允许用r''表示''内部的字符串默认不转义 & Python允许用'''...'''的格式表示多行内容  
4.布尔值：True、False（请注意大小写）  
5.空值：None  
6.变量：a = 'ABC'时，Python解释器干了两件事情：  
　　在内存中创建了一个'ABC'的字符串；  
　　在内存中创建了一个名为a的变量，并把它指向'ABC'    
7.list:[]有序集合，用-1做索引，直接获取最后一个元素  
　　len(list):取长度  
　　list.append(a)追加  
　　list.insert(1，a)元素插入  
　　list.pop()删除list末尾的元素  
　　list.pop(i)删除置顶位置的元素  
　　li =  [1, 2, 4, 3]  
　　(It's a closed/open range for you mathy types.)  
　　li[1:3]  # => [2, 4]  
 　　Omit the beginning  
　　li[2:]  # => [4, 3]  
　　 Omit the end  
　　li[:3]  # => [1, 2, 4]  
 　　Select every second entry  
　　li[::2]  # =>[1, 4]  
 　　Reverse a copy of the list  
　　li[::-1]  # => [3, 4, 2, 1]  
  
8.tuple:()有序列表，一旦初始化后不能修改  Python在显示只有1个元素的tuple时，也会加一个逗号,，以免你误解成数学计算意义上的括号(1,)  
9.dict:dict全称dictionary，在其他语言中也称为map，使用键-值（key-value）存储，具有极快的查找速度  
　　dict提供的get方法，如果key不存在，可以返回None，或者自己指定的value：d.get('Thomas', -1)  
　　d.pop(key):删除key  
10.set：在set中，没有重复的key  
　　s.add(key)  
　　s.remove(key)  

四、函数  
　　绝对值函数：abs(x)  
　　帮助命令：help(abs)  
　　比较函数：cmp(x,y)  
　　转换:int()、float()、str()、unicode()、bool()  
　　类型检查：isinstance（）

五、函数的参数  
　　要注意定义可变参数和关键字参数的语法：  
　　*args是可变参数，args接收的是一个tuple；  
　　**kw是关键字参数，kw接收的是一个dict。  
　　参数组合：参数定义的顺序必须是：必选参数、默认参数、可变参数和关键字参数。
 
六、递归函数  
　　使用递归函数需要注意防止栈溢出。在计算机中，函数调用是通过栈（stack）这种数据结构实现的，每当进入一个函数调用，栈就会加一层栈帧，每当函数返回，栈就会减一层栈帧。由于栈的大小不是无限的，所以，递归调用的次数过多，会导致栈溢出。   
　　解决递归调用栈溢出的方法是通过尾递归优化，事实上尾递归和循环的效果是一样的，所以，把循环看成是一种特殊的尾递归函数也是可以的  
　　尾递归是指，在函数返回的时候，调用自身本身，并且，return语句不能包含表达式。这样，编译器或者解释器就可以把尾递归做优化，使递归本身无论调用多少次，都只占用一个栈帧，不会出现栈溢出的情况。

七、其他  
	　　\_\_future\_\_：Python提供了\_\_future\_\_模块，把下一个新版本的特性导入到当前版本，于是我们就可以在当前版本中测试一些新版本的特性。
