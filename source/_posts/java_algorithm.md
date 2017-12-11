---

title: java常用算法
data: 2017-11-29
categories: algorithm,java
tags: [algorithm,java]

---

- 排序和查找算法  
- 单链表  
- 二叉树  
- 队列和栈  
- 字符串  
- 数组  
- 其它算法  


在排序中，90%的概率会考察快速排序算法，一些简单的时候，会要求二分查找算法

快速排序：是一种分区交换排序算法。采用分治策略对两个子序列再分别进行快速排序，是一种递归算法。  
算法描述：在数据序列中选择一个元素作为基准值，每趟从数据序列的两端开始交替进行，将小于基准值的元素交换到序列前端，将大于基准值的元素交换到序列后端，介于两者之间的位置则成为了基准值的最终位置。同时，序列被划分成两个子序列，再分别对两个子序列进行快速排序，直到子序列的长度为1，则完成排序。

举例：假设要排序的数组是a[6],长度为7，首先任意选取一个数据（通常选用第一个数据）作为关键数据，然后将所有比它的数都放到它前面，所有比它大的数都放到它后面，这个过程称为一躺快速排序。一躺快速排序的算法是：  

设置两个变量i，j，排序开始的时候i=0；j=6； 
 
- 以第一个数组元素作为关键数据，赋值给key，即key=a[0]；  
- 从j开始向前搜索，即由后开始向前搜索（j--），找到第一个小于key的值，两者交换；  
- 从i开始向后搜索，即由前开始向后搜索（i++），找到第一个大于key的值，两者交换；  
- 重复第3、4步，直到i=j；此时将key赋值给a[i]；  

例如：待排序的数组a的值分别是：（初始关键数据key=49）

![](http://images.gitbook.cn/07827ec0-bd77-11e7-a4d0-01a65989f9d2)

此时完成了一趟循环，将49赋值给a[3]，数据分为三组，分别为{27,38,13}{49}{76,96,65}，利用递归，分别对第一组和第三组进行排序，则可得到一个有序序列，这就是快速排序算法。
快速排序代码如下

	public void quickSort(int[] num, int left, int right) { 
    	if (num == null) return; //如果左边大于右边，则return，这里是递归的终点，需要写在前面。 
    	if (left >= right) { return; } 
    	int i = left; 
    	int j = right;
     	int temp = num[i]; //此处开始进入遍历循环 
    	while (i < j) { //从右往左循环 
    		while (i < j && num[j] >= temp) {//如果num[j]大于temp值，则pass，比较下一个 
    			j--; } 
    		num[i] = num[j]; 
   			while (i < j && num[i] <= temp) { 
				i++; } 
    		num[j] = num[i]; 
			num[i] = temp; // 此处不可遗漏，将基准值插入到指定位置 
    	} 
		quickSort(num, left, i - 1); 
    	quickSort(num, i + 1, right); 
	}

**二分查找又称折半查找**，优点是比较次数少，查找速度快，平均性能好；其缺点是要求待查表为有序表，且插入删除困难。因此，折半查找方法适用于不经常变动而查找频繁的有序列表。  

步骤：首先，假设表中元素是按升序排列，将表中间位置记录的关键字与查找关键字比较，如果两者相等，则查找成功；否则利用中间位置记录将表分成前、后两个子表，如果中间位置记录的关键字大于查找关键字，则进一步查找前一子表，否则进一步查找后一子表。重复以上过程，直到找到满足条件的记录，使查找成功，或直到子表不存在为止，此时查找不成功。

算法前提：必须采用顺序存储结构；必须按关键字大小有序排列。  

实现方式：包含递归实现和非递归实现两种方式。二分查找的递归代码实现如下：  
	
	private  int halfSearch(int[] a,int left,int right,int target) {  
     int mid=(left+right)/2;  
     int midValue=a[mid];  
     if(left<=right){  
         if(midValue>target){  
             return halfSearch(a, left, mid-1, target);  
         }else if(midValue<target) {  
             return halfSearch(a, mid+1, right, target);  
         }else {  
             return mid;  
         }  
    }  
    return -1;  
	}

非递归实现代码如下：

	private  int halfSearch(int[] a,int target){  
    int i=0;  
    int j=a.length-1;  
    while(i<=j){  
        int mid=(i+j)/2;  
        int midValue=a[mid];  
        if(midValue>target){  
            j=mid-1;  
        }else if(midValue<target){  
            i=mid+1;  
        }else {  
            return mid;  
        }  
    }  
    return -1;  
	}

[更多排序查找算法](http://blog.csdn.net/qq_25827845/article/details/74058248)