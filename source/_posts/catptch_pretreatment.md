---

title: 验证码图片预处理—python 
data: 2017-11-16
categories: python,Machine Learning
tags: [python,Machine Learning,captcha]

---

本文主要讨论下验证码识别之前一般都会做的验证码的预处理

验证码的预处理一般有一下几个步骤  
- 图片去噪（去干扰点及干扰线）  
- 将彩色图片二值化为黑白图片  
- 图片字符切割    
- 图片尺寸归一化  
- 图片字符标记

以上的步骤可以根据实际情况做先后顺序的调整


   
**图片去噪**  
- 去噪点待补充  参考http://blog.csdn.net/ysc6688/article/details/50772382  
- 去干扰线待补充  参考http://blog.csdn.net/ysc6688/article/details/44540623  
  

**二值化图片**   
主要步骤如下：
将RGB彩图转为灰度图   
将灰度图按照设定阈值转化为二值图

一、灰度化   
灰度化应用很广，而且也比较简单。灰度图就是将白与黑中间的颜色等分为若干等级，绝大多数位256阶。在RGB模型种，黑色（R=G=B=0）与白色（R=G=B=255），那么256阶的灰度划分就是R=G=B=i，其中i取0到255.   
一般图像的颜色数据矩阵默认是3通道的，也就是RGB模型，所以每个pixel都有3个分量，分别代表r，g和b的值。因此将三个分量值都改为同一个灰度值，图片就实现灰度化。     
灰度化的方法一般有以下几种：   
1. 分量法    
在rgb三个分量种按照需求选取一个分量作为灰度值   
2. 最大值    
选取rgb的最大值作为该pixel的灰度值   
3. 平均值    
g[i,j] = (r[i,j] + g[i,j] + b[i,j]) / 3,取rgb的平均值作为灰度值
4. 加权变换    
由于人眼对绿色的敏感最高，对蓝色敏感最低，因此，按下式对RGB三分量进行加权平均能得到较合理的灰度图像。    L = R * 299/1000 + G * 587/1000 + B * 114/1000  
*PIL 包中的Image.convert('L')  就是采用的这种计算方式*

灰度后的图片如下

![9639_L.png](https://i.loli.net/2017/11/16/5a0d523089f32.png)


二、二值化  
二值化故名思议，就是整个图像所有像素只有两个值可以选择，一个是黑（灰度为0），一个是白（灰度为255）。二值化的好处就是将图片上的有用信息和无用信息区分开来，比如二值化之后的验证码图片，验证码像素为黑色，背景和干扰点为白色，这样后面对验证码像素处理的时候就会很方便。   
常见的二值化方法为固定阀值和自适应阀值，固定阀值就是制定一个固定的数值作为分界点，大于这个阀值的像素就设为255，小于该阀值就设为0，这种方法简单粗暴，但是效果不一定好.另外就是自适应阀值，每次根据图片的灰度情况找合适的阀值。自适应阀值的方法有很多，这里采用了一种类似K均值的方法，就是先选择一个值作为阀值，统计大于这个阀值的所有像素的灰度平均值和小于这个阀值的所有像素的灰度平均值，再求这两个值的平均值作为新的阀值。重复上面的计算，直到每次更新阀值后，大于该阀值和小于该阀值的像素数目不变为止。
python代码如下：

    #获取一个图片的自适应阈值
	def adaptiveThreshold(image):
      	img_raw =  np.array(image)
    	width, height = image.size
    	ucThre = 0;  
    	ucThre_new = 127;#该值初始不等于ucThre即可  
  
    	while ucThre != ucThre_new:
        	nBack_count = 0
        	nData_count = 0   
        	nBack_sum = 0
        	nData_sum = 0  
        	for j in  range(height):
            	for i in range(width):
                	nValue = img_raw[j][i]  
                	if nValue > ucThre_new:  
                    	nBack_sum += nValue
                    	nBack_count+=1
                	else:  
                    	nData_sum += nValue
                    	nData_count+=1  
        	nBack_sum = nBack_sum / nBack_count
        	nData_sum = nData_sum / nData_count
        	ucThre = ucThre_new
        	ucThre_new = (nBack_sum + nData_sum) / 2        	    
    	return math.ceil(ucThre_new)
  
定义二值函数：  

    def get_bin_table(threshold=230):
    	table = []
    	for i in range(256):
    		if i < threshold:
    			table.append(0)
    		else:
    			table.append(1)
     
    	return table


完整将一个图片二值化的过程如下：  

    image = Image.open(img_path)
    imgry = image.convert('L')  # 转化为灰度图    
    table = get_bin_table(adaptiveThreshold(imgry))
    out = imgry.point(table, '1')

经过上述转化后的二值化图片如下   
![9639_bina.png](https://i.loli.net/2017/11/16/5a0d52533f8f5.png)



**图片字符切割**  

前面经过各种去除噪点、干扰线，验证码图片现在已经只有两个部分，白色背景及黑色字符。为了字符的识别，这里需要将图片上的字符一个一个“扣”下来，得到单个的字符，接下来再进行识别。  
字符分割可以说是图像验证码识别最关键的一步，因为分割的正确与否直接关系到最后的结果，如果4个字符分割成了3个，即便后面的识别算法识别率达到100%，结果也是错的。当然，前面预处理如果做得够好，干扰因素能够有效的去除，而没有影响到字符的像素，那么分割来讲要容易得多。反过来，如果前面的干扰因素都没有去除掉，那么分割出来的可能就不是字符了。  
字符的粘连是分割的难点，这一点也可以作为验证码安全系数的标准，如果验证码上的几个字符完全是分开的，那么可以保证字符分割成功率百分之百，这样验证码破解的难度就降低了很多
上述处理过的简单示例就是一个没有黏连的验证码  
可以有2中简单的处理方式   

一、洪水填充法   

洪水填充法主要思路还是连通域的思想。对于相互之间没有粘连的字符验证码，直接对图片进行扫描，遇到一个黑的pixel就对其进行填充，所有与其连通的字符都被标记出来，因此一个独立的字符就能够找到了。这个方法优点是效率高，时间复杂度是O（N），N为像素的个数；而且不用考虑图片的大小、相邻字符间隔以及字符在图片中得位置等其他任何因素，任何验证码图片只要字符相互是独立的，不需要对其他任何阀值做预处理，直接就操作；用这种方法分割正确率非常高，几乎不会出现分割错误的情况。但是缺点也很致命：那就是字符之间必须完全隔离，没有粘连的部分，否则会将两个字符误认为一个字符。同时若有不相连的字母如i,j也不太好处理


二、X像素投影法

对于粘连的字符，也并非没有方法分割。一个方法就是将两个粘连的验证码一刀切开，从哪里切？当然是从粘连的薄弱的地方切。前面提到过图片的像素就像一个二维的矩阵，对每一个x值，统计所有x值为这个值的pixel中黑色的数目，直观来讲就是统计每一条竖线上黑色点的数目。显而易见的是，如果这一条线为背景，那么这一条线肯定都是白色的，那么黑色点的数目为0，如果一条竖线经过字符，那么这条竖线上的黑色点数目肯定不少。  
对于完全独立的两个字符之间，肯定有黑色点数目为0的竖线，但是如果粘连，那么不会有黑色点数为0的竖线存在，但是字符粘连最薄弱的地方一定是黑色点数目最少的那条竖线，因此切就要从这个地方切。  
在代码的实现的过程中，可以先从左到右扫描一遍，统计投影到每个X值的黑色点的数目，然后设定一个阀值范围，这个阀值大概就是一个字符的宽度。从左到右，先找到第一个x黑色点投影不为0的x值，然后在这个x值加上大概一个字符宽度的大小找到x投影数目最小的x值，这两个x值分割出来就是一个字符了。  
这个方法的特点就是能够分割粘连的字符，但是缺点就是容易分割不干净，可能会出现分割错误的情况，另外就是需要提供相应的阀值。如果验证码字体倾斜比较厉害，就不可以使用，比如
![20150323195933323.jpg](https://i.loli.net/2017/11/16/5a0d526c41e22.jpg)

    # 图像处理-X像素投影法,0像素切割
	def splitByPixel(image, text):
    # img_raw = np.array(image)
    	w, h = image.size
    	splitCol = getSplit(image)

    	if len(splitCol) != len(text):
        	print("切割错误！", text)
        	return ""

    	for i in range(len(text)):
        	cropImg = image.crop((splitCol[i][0], 0, splitCol[i][1], h))
        	cropImg = resizeImge(cropImg, int(w / int(len(text))), h)  # 切割的图片尺寸不一，统一尺寸
        	char = text[i]
        	save_path = "./knnImage/letters/" + str(char) + "/" + str(uuid.uuid4()) + ".png"
        	cropImg.save(save_path)  # 保存切割下来的图片
	

	# 获取切割列0像素切割
	def getSplit(image):
    	img_raw = np.array(image)
    	w, h = image.size
    	allCol = []
    	for i in range(w):
        	allCol.append(i)
    	charCol = []
    	for i in range(w):
        	for j in range(h):
            	if img_raw[j][i] != True:
                	charCol.append(i)
                	break
    	# 找到0像素的列
    	zeroCol = [i for i in allCol if i not in charCol]
    	# 从0像素列中找出切割列
    	splitCol = []
    	col = zeroCol[1]
    	for vi, vv in enumerate(zeroCol):
        	tmparr = []
        	if vv < col + 4:
            	col += 1
        	else:
            	if vi > 0:
                	tmparr.append(zeroCol[vi - 1])
            	else:
                	tmparr.append(zeroCol[0])
            	tmparr.append(vv)
            	splitCol.append(tmparr)
            	col = vv

    	return splitCol
	


**图片尺寸归一化**  
在图片切割的时候，就将切割完的图片统一尺寸
	
	def resizeImge(image,rew,reh):
    	img_raw = np.array(image)
    	w, h = image.size
    	top = 0
    	bottom =h
    	try:
        	for i in range (h):
            	for j in range(w):
                	if img_raw[i][j] != True:
                    	top = i - 1
                    	raise Getoutofloop()
    	except Getoutofloop:
        	pass

    	try:
        	for i in range(h):
            	for j in range(w):
                	if img_raw[h-1-i][j] != True:
                    	bottom = h - i
                    	raise Getoutofloop()
    	except Getoutofloop:
        	pass

    	cropImg = image.crop((0, top, w, bottom))
    	cropImg= cropImg.resize((rew,reh))

    	return cropImg

	class Getoutofloop(Exception):
    	pass

**图片字符标记**
即给验证码标注正确的字符


到此验证码的预处理工作就大概完成了