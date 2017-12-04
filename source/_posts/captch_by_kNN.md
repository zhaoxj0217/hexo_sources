---

title: 基于K近邻算法的验证码识别—knn 
data: 2017-11-02
categories: python,KNN,Machine Learning
tags: [python,KNN,Machine Learning,captcha]

---


本文主要基于K近邻算法进行验证码识别，是验证码识别中最简单的一种机器学习方法

**1.1、什么是K近邻算法**

&emsp;&emsp;所谓K近邻算法，即是给定一个训练数据集，对新的输入实例，在训练数据集中找到与该实例最邻近的K个实例（也就是上面所说的K个邻居），这K个实例的多数属于某个类，就把该输入实例分类到这个类中。
咱们来看下引自维基百科上的一幅图：   
![knn_pic1.png](http://img.my.csdn.net/uploads/201211/20/1353395335_6987.png)   

 &emsp;&emsp;如上图所示，有两类不同的样本数据，分别用蓝色的小正方形和红色的小三角形表示，而图正中间的那个绿色的圆所标示的数据则是待分类的数据。我们就要解决这个问题：给这个绿色的圆分类。      
 &emsp;&emsp;如果K=3，绿色圆点的最近的3个邻居是2个红色小三角形和1个蓝色小正方形，少数从属于多数，基于统计的方法，判定绿色的这个待分类点属于红色的三角形一类。  
 &emsp;&emsp;如果K=5，绿色圆点的最近的5个邻居是2个红色三角形和3个蓝色的正方形，还是少数从属于多数，基于统计的方法，判定绿色的这个待分类点属于蓝色的正方形一类。  
  &emsp;&emsp;于此我们看到，当无法判定当前待分类点是从属于已知分类中的哪一类时，我们可以依据统计学的理论看它所处的位置特征，衡量它周围邻居的权重，而把它归为(或分配)到权重更大的那一类。这就是K近邻算法的核心思想。

**近邻的距离度量表示法**

&emsp;&emsp;K近邻算法的核心在于找到实例点的邻居，这个时候，问题就是如何找到邻居，邻居的判定标准是什么，用什么来度量。特征空间中两个实例点的距离可以反应出两个实例点之间的相似性程度，K近邻模型的特征空间一般是n维实数向量空间，使用的距离可以使最常用的欧式距离，也是可以是其它距离（汉明距离，闵可夫斯基距离，切比雪夫距离，曼哈顿距离，这里又可以扩展到范数的概念）

**K值的选择**

除了上述如何定义邻居的问题之外，还有一个选择多少个邻居，即K值定义为多大的问题。不要小看了这个K值选择问题，因为它对K近邻算法的结果会产生重大影响。如李航博士的一书「统计学习方法」上所说：
如果选择较小的K值，就相当于用较小的领域中的训练实例进行预测，“学习”近似误差会减小，只有与输入实例较近或相似的训练实例才会对预测结果起作用，与此同时带来的问题是“学习”的估计误差会增大，换句话说，K值的减小就意味着整体模型变得复杂，容易发生过拟合；
如果选择较大的K值，就相当于用较大领域中的训练实例进行预测，其优点是可以减少学习的估计误差，但缺点是学习的近似误差会增大。这时候，与输入实例较远（不相似的）训练实例也会对预测器作用，使预测发生错误，且K值的增大就意味着整体的模型变得简单。
K=N，则完全不足取，因为此时无论输入实例是什么，都只是简单的预测它属于在训练实例中最多的累，模型过于简单，忽略了训练实例中大量有用信息。
在实际应用中，K值一般取一个比较小的数值，例如采用交叉验证法（简单来说，就是一部分样本做训练集，一部分做测试集）来选择最优的K值。



**基本流程**  
一般情况下，对于字符型验证码的识别流程如下：
   
- 准备原始验证码素材  
- 图片预处理
- 生成训练数据集  
- 训练模型（KNN不涉及） 
- 测试验证码识别率 


 **素材选择**   
选择需要识别的验证码，验证码的复杂程度不影响上述的基本流程，所以选择一个比较简单的验证码进行示范

![9639.jpg](https://i.loli.net/2017/11/16/5a0d51e2715c3.jpg)

该验证码由4位阿拉伯数组组成，没有干扰线，没有背景噪点  
因机器学习过程需要大量的样本，样本的获取可以去目标网站获取，该方式的缺点是需要手动标记，另一种是自己用代码批量生成，前提是你已了解了对方验证码生成的方式（或者近似的方式）


**图片预处理**

为了减少后面训练时的复杂度，同时增加识别率，很有必要对图片进行预处理，使其对机器识别更友好。

图片预处理的过程请参考站内另一文章： 验证码图片预处理—python

**生成训练数据集**  
	

将分割完毕的字符并标记好的样本转化为数学向量导入内存中作为样本数据

    def createDataSet():
    	print("start init train data...")
    	global group
    	global lables
    
    	for number in range(10):
    		im_paths = filter(lambda fn: os.path.splitext(fn)[1].lower() == '.png',os.listdir("./knnImage/letters/"+str(number)))
    
    	for im_path in im_paths:
    		imagepath = "./knnImage/letters/" +str(number)+"/"+ im_path
    		image = Image.open(imagepath)
    		try:
    			group.append(img2vector(image))
    			lables.append(number)
    		except:
    			print(im_path,"createDataSetError")
    		pass
    #转为numpy数组
    	group = np.array(group)
    	lables = np.array(lables)
    	print("end init train data...")

	#使用到的图片转向量函数
	#图片转向量
	def img2vector(image):
    	returnVect =[]
    	img_raw = np.array(image)
    	width, height = image.size
    	for h in range(height):
        	for w in range(width):
            	if img_raw[h][w] ==True:#因为二值化图片的结果是True跟False,这里转化为01
                	returnVect.append(0)
            	else:
                	returnVect.append(1)
    	return returnVect

**训练模型**    
knn算法不涉及模型训练

**验证码识别**

 识别验证码  

    def getCaptcha(image):
    	# img_raw = np.array(image)
    	global group
    	global lables
    	w, h = image.size
    	splitCol = getSplit(image)
    
    	if len(splitCol) != 4:
    		print("切割错误！", text)
    		return ""
    
    	chars = []
    	for i in range(4):
    		cropImg = image.crop((splitCol[i][0], 0, splitCol[i][1], h))
    		cropImg = resizeImge(cropImg, int(w /4), h)  # 切割的图片尺寸不一，统一尺寸
    		charvec = img2vector(cropImg)
    		char =  classify(charvec, group, lables, 3)
    		chars.append(str(char))
    	return "".join(chars)

    


**knn算法**

    # knn分类算法 此处使用欧式距离公式
	#inX :待分类向量
	#dataSet：训练集合
	#labels：训练集合对应的分类结果
	#k值
	def classify(inX, dataSet, labels, k):
    	# 距离计算 start,
    	dataSetSize = dataSet.shape[0]# 计算有多少个训练样本
    	tmp = np.tile(inX, (dataSetSize, 1))# 将待分类的输入向量进行 行反向复制dataSetSize次，列方向复制1一次,即与训练样本大小一致
    	diffMat = tmp - dataSet # 数组相减
    	sqDiffMat = diffMat ** 2 #2次方
    	sqDistances = sqDiffMat.sum(axis=1)# 对于二维数组axis=1表示按行相加 , axis=0表示按列相加
    	distances = sqDistances ** 0.5 # 平方根
    	# 距离计算 end
    	sortedDistIndicies = distances.argsort() # 将输入与训练集的距离排序 argsort函数返回的是数组值从小到大的索引值
    	classCount = {}
    	for i in range(k): # 统计距离最近的K个lable值
        	voteIlabel = labels[sortedDistIndicies[i]]
        	classCount[voteIlabel] = classCount.get(voteIlabel, 0) + 1
    	sortedClassCount = sorted(classCount.items(), key=lambda item: item[1], reverse=True)
    	return sortedClassCount[0][0]#返回k个中数量最多的label


测试knn分类的效果

    if __name__ == "__main__":
       
    # 图片识别  --start
    #创建训练集合
    	createDataSet()#训练数据加载比较慢
    
    	im_paths = filter(lambda fn: os.path.splitext(fn)[1].lower() == '.jpg',os.listdir("./knnImage/test"))
    
    	total = 0
    	right = 0
    	for im_path in im_paths:
    		total+=1
         	imagepath = "./knnImage/test/" + im_path
    		image = Image.open(imagepath)
    		imgry = image.convert('L')  # 转化为灰度图
    	    table = get_bin_table(230)
    		out = imgry.point(table, '1')
    		text = os.path.basename(im_path).split('_')[0]
    		knntext = getCaptcha(out)
    		if text == knntext:
   				right+=1
    			print("knn right")
    		else:
    			print(text,"knn error:",knntext)
    	print("right:total is:",right,total)    
    # 图片识别  --end


如果测试结果能满足预期的识别率，就可以使用该训练样本去实际使用了
