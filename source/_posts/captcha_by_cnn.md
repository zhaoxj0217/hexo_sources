---

title: 基于tensorflow的‘端到端’的字符型验证码识别—cnn 
data: 2017-11-02
categories: python,CNN,tensorflow,
tags: [python,CNN,深度学习,tensorflow,captcha]

---

因有个验证码识别的需求，本来打算用SVM来识别验证码，但在查询资料的过程中看到了@斗大的熊猫
[TensorFlow练习20: 使用深度学习破解字符验证码](http://blog.topspeedsnail.com/archives/10858)
这篇文章
在github上也找到了基于这篇文章的代码整合  
[基于tensorflow的‘端到端’的字符型验证码识别](https://github.com/zhengwh/captcha-tensorflow)

因为传统的机器学习方法，对于多位字符验证码都是采用的化整为零的方法：先分割成最小单位，再分别识别，然后再统一。同时还需对图片进行去躁，去干扰线等预处理，预处理的好坏直接影响识别率，所以想看下
卷积神经网络方法，是否真的更加通用简单，其他相关的说明都在这2个连接中

本文的代码是在win7,python35环境下跑的
针对captcha-tensorflow的代码做了一些修改（主要是数据读取的方式的修改，原来的代码是直接代码生成验证码图片进行训练，本文需要训练的验证码是通过其他方式批量生成后再进行训练）

修改后的代码目录与原来代码目录基本一致（去掉了没使用的验证码生成）

    capt
		__init__.py
		cfg.py  配置信息文件
		cnn_sys.py  CNN网络结构
		data_iter.py 可迭代的数据集
		predict.py 加载训练好的模型，然后对输入的图片进行预测
		train.py 对模型进行训练
		utils.py 一些公共使用的方法
	tmp
	work

cfg.py就根据实际情况配置下  
cnn_sys.py 需要根据你视图验证码的图像大小，做下计算参数上的小调整  
predict.py 根据需要修改获取验证数据方式  
train.py  有个须知的地方：
	
    # 从0开始训练数据
    # sess.run(tf.global_variables_initializer())

    #在训练一段时间后接着上次的训练
    saver.restore(sess, tf.train.latest_checkpoint(model_path))


data_iter.py 提供数据与原来的方法有所区别:  
    
	"""
	数据生成器
	"""
	import numpy as np

	from capt.cfg import IMAGE_HEIGHT, IMAGE_WIDTH, CHAR_SET_LEN, MAX_CAPTCHA,trainSpace,testSpace,verifySpace
	from capt.utils import convert2gray, text2vec
	from tensorflow.python.platform import gfile
	import os.path
	from PIL import Image
	import random
	from os.path import join

	no1 = 0
	no2 = 1
	dataSet = []
	testdataSet = []
	verifydataSet =[]

	#从相应文件夹内读取验证码图片
	def create_data_list(image_dir):
		if not gfile.Exists(image_dir):
    		print("Image director '" + image_dir + "' not found.")
    		return None
  
  		extensions = ['jpg']
  		print("Looking for images in '" + image_dir + "'")
  		file_list = []
  		for extension in extensions:
    		file_glob = os.path.join(image_dir, '*.' + extension)
    		file_list.extend(gfile.Glob(file_glob))
  		if not file_list:
    		print("No files found in '" + image_dir + "'")
    		return None

  		imageList = []
  		for file_name in file_list:
    		images = []
    		image = Image.open(file_name)
    		img_raw =  np.array(image)
    		image.close()
    		label_name = os.path.basename(file_name).split('_')[0]
    		images.append(img_raw)
    		images.append(label_name)
    		imageList.append(images)
  		print("create imgList finish!")
  		return imageList


	#生成训练数据
	def get_train_batch(batch_size=128):
   		#因为待训练的图片很多（几十万张）一次读取很占内存，将他们分成1万一个子文件夹
    	batch_x = np.zeros([batch_size, IMAGE_HEIGHT * IMAGE_WIDTH])
    	batch_y = np.zeros([batch_size, MAX_CAPTCHA * CHAR_SET_LEN])

    	global no1
    	no1= no1+1
    	print('no1',no1)
    	global no2
    	global dataSet
    	image_dir = join(trainSpace, no2)
    	if(no1>n):#单个子集训练n个step
       		no1=0
        	no2 =no2+1
        	if (no2 > j):#全部子集训练完后循环训练
           		no2 = 1
        	image_dir = join(trainSpace, no2)
        	dataSet = create_data_list(image_dir)
        	print("create trainSET")
        	print('image_dir', image_dir)
    	print('no2', no2)
    	if(not dataSet):
        	dataSet = create_data_list(image_dir)
        	print("create trainSET")
        	print('image_dir', image_dir)

    	for i in range(batch_size):
        	data = dataSet[random.randint(0, len(dataSet)-1)]#随机读取数据
        	image = data[0]
        	text = data[1]
        	image = convert2gray(image)
        	batch_x[i, :] = image.flatten() / 255  # (image.flatten()-128)/128  mean为0
        	batch_y[i, :] = text2vec(text)

    	return batch_x, batch_y

	#获取测试数据
	def get_test_batch(batch_size=128):
    
    	batch_x = np.zeros([batch_size, IMAGE_HEIGHT * IMAGE_WIDTH])
    	batch_y = np.zeros([batch_size, MAX_CAPTCHA * CHAR_SET_LEN])

    	global testdataSet
    	if(not testdataSet):
       		image_dir = testSpace
        	testdataSet = create_data_list(image_dir)
        	print("create testSET")

    	for i in range(batch_size):
        	data = testdataSet[random.randint(0, len(testdataSet)-1)]
        	image = data[0]
        	text = data[1]
        	image = convert2gray(image)
        	batch_x[i, :] = image.flatten() / 255  # (image.flatten()-128)/128  mean为0
        	batch_y[i, :] = text2vec(text)

    	return batch_x, batch_y

	#获取验证数据
	def get_verify():
    	global verifydataSet
    	if(not verifydataSet):
        	image_dir = "D:\\tmp\\cnnImage\\verify4"
        	verifydataSet = create_data_list(image_dir)
        	print("create verifySET")

    	data = verifydataSet[random.randint(0, len(verifydataSet)-1)]
    	image = data[0]
    	text = data[1]
    	return  text,image

在监控文件所在盘 执行tensorboard --logdir=...
(这个cmd 命令与在百度上查询到的在linux上执行的不一样，需要注意)
就可以在tensorboard上查看监控图像，

本次训练在经过2万个step 以后ACC达到95%，识别准确率在85%

![](http://chuantu.biz/t6/122/1509618243x3663627938.png)






