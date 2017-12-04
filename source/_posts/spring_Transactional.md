---

title: 深入理解spring事物机制—transactional 
data: 2017-11-06
categories: spring,transactional,mysql
tags: [spring]

---

## 事务是什么？
**&emsp;- 狭义上的理解：关系型数据库事务   
&emsp;- ACID：原子性、一致性、隔离性、持久性   
&emsp;- 为什么我们要用“事务”？   
&emsp;- 测试的盲点**  

**首先问几个问题**

*问题一：*

    @Transactional(propagation = Propagation.REQUIRED)
    A.test();

    @Transactional(propagation = Propagation.REQUIRED)
    B.test();

    @Transactional(propagation = Propagation.REQUIRED)
    Entry.test(){
        try{
            A.test();
        }catch(Exception e){
            //这里捕获异常会回滚Entry.test事物么？
        }
        B.test();
    }

*问题二：*

	@Transactional(propagation = Propagation.REQUIRES_NEW)
    A.test();

    @Transactional(propagation = Propagation.REQUIRED)
    B.test();

    @Transactional(propagation = Propagation.REQUIRED)
    Entry.test(){
        try{
            A.test();
        }catch(Exception e){
            //这里捕获异常会回滚Entry.test事物么？
        }
        B.test();
    }

*问题三：*

	@Transactional(propagation = Propagation.REQUIRES_NEW)
    A.test();

    @Transactional(propagation = Propagation.REQUIRED)
    B.test();

    @Transactional(propagation = Propagation.REQUIRED)
    Entry.test(){
 		A.test();
		B.test();
        try{
           throw new RuntimeException()
        }catch(Exception e){
            //这里捕获异常会回滚Entry.test事物么？
        }
        
    }

*问题四：*

	public class A {

    @Transactional(propagation = Propagation.REQUIRED)
    public void testA1(){
        try{
            this.testA2();
        }catch (Exception e){
            //这里捕获异常会回滚testA1的事务么
        }
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public void testA2(){
    }
	}

*问题五：*

	@Transactional(timeout = 5)
    Entry.test(){
        Thread.sleep(6000);
        A.test();//执行数据库操作，耗时1秒
    }
    //超时会导致Entry.test事务回滚么？

*问题六：*
	
	@Transactional(timeout = 5)
    Entry.test(){       
        A.test();//执行数据库操作，耗时1秒
 		Thread.sleep(6000);
    }
    //超时会导致Entry.test事务回滚么？

*问题七：*

	@Transactional()
    Entry.test(){       
        A.test();//执行数据库操作，在并发情况下会争夺行锁
    }
    //为什么没有配置超时，却还是捕捉到了超时异常：
	###Cause:java.sql.SQLException:Lock wait timeout exceeded; try restarting transaction

*问题八：*  
	&emsp;@Transactional()应该加在实现类的方法上还是接口的方法上?

*问题九：*  
	&emsp;@Transactional()加在private、final、protected方法上有用吗?


## 事务的代码流转 
&emsp;&emsp;spring事务管理器->mybatis->jdbc->mysql


MySQL（InnoDB）对事务的支持:  
&emsp;begin:   
&emsp;savepoint:  
&emsp;rollback:  
&emsp;commit:  

MySQL（InnoDB）不支持事务嵌套，第二个begin会隐式地把第一个给先commit掉

Jdbc对应的api

    java.sql.Driver.java:
    	-Connection connect(String url, java.util.Properties info)throws SQLException; //获取数据库连接
    
    java.sql.Connection.java:
    	-void setAutoCommit(boolean autoCommit) throws SQLException;//设置为false表示事务begin
    	-Savepoint setSavepoint() throws SQLException; //创建保存点Savepoint 
    	-setSavepoint(String name) throws SQLException; //创建保存点void 
    	-rollback(Savepoint savepoint) throws SQLException; //回滚保存点void 
    	-rollback() throws SQLException; //回滚整个事务void 
    	-releaseSavepoint(Savepoint savepoint) throws SQLException;//删除保存点
    	-void commit() throws SQLException; //提交整个事务Statement 
    	-createStatement() throws SQLException;//创建语句
    
    java.sql.Statement.java:
    	-int executeUpdate(String sql) throws SQLException;//执行sql更新语句，当然还有其他执行接口

Savepoint的操作要jdbc3.0及以上才支持


**mysql 事务的三大属性**  

- 隔离级别  <table>
        <tr>
            <th>隔离级别</th>
            <th>脏度可能性</th>
            <th>不可读可能性</th>
            <th>幻读可能性</th>
        </tr>
        <tr>
            <th>READ UNCOMMITTED</th>
            <th>是</th>
            <th>是</th>
            <th>是</th>
        </tr>
        <tr>
            <th>READ COMMITTED</th>
            <th>否</th>
            <th>是</th>
            <th>是</th>
        </tr>
        <tr>
            <th>REPEATABLE READ</th>
            <th>否</th>
            <th>否</th>
            <th>是</th>
        </tr>
		<tr>
            <th>SERIALIZABLE</th>
            <th>否</th>
            <th>否</th>
            <th>否</th>
        </tr>
   </table>

Mysql默认隔离级别是REPEATABLE READ
查看命令：select @@tx_isolation

>脏读：读取的是别的事务的半成品，不会用于生产环境。  
>
>不可重复读：是指在一个事务内，多次读同一行数据。在这个事务还没有结束时，另外一个事务修改该行数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改，那么第一个事务两次读到的的数据可能是不一样的。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不可重复读。
>
>可重复读：就是不可重复读的取反
>
>幻读:是指在一个事务内，按条件读取几行数据。在这个事务还没有结束时，另外一个事务插入或者删除了符合条件的数据。那么，在第一个事务中的两次相同条件的读取数据之间，由于第二个事务的插入和删除，那么第一个事务两次读到的的数据行数可能是不一样的。
>>幻读对代码编写方式的影响  
>>  CRIRATE TABLE 'tb_trans_student'(  
>>    'Id' bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT 'Id'，  
>>    'Name' varchar(255) NOT NULL COMMENT '姓名'，   
>>    'Score' smallint(3) unsigned NOT NULL COMMENT '分数'，  
>>    PRIMARY KEY ('Id')，  
>>    UNIQUE KEY 'Name' ('Name')  
>>  )ENGINE = InnoBD AUTO_INCREMENT =18 DEFAULT CHARSET =utf8 COMMENT ='学生表'  
>>

	@Transactional
	public void insert(Object data){
		try{
			Object dataInDB = db.selectWhere(data.name);	
			if(dataInDB== null){
				db.insert(data);//另一个线程刚刚插入，导致这里会违反唯一性约束
			}
		}catch(Exception e){
			Object dataInDB = db.selectWhere(data.name);
			log("数据库已存在"+dataInDB);//这里dataInDB为null
		}	
	}
>>插又插不进去，读也读不到

**Jdbc对应的api(事务隔离级别)**  
- java.sql.Connection.java:  
- int TRANSACTION_READ_UNCOMMITTED = 1;  
- int TRANSACTION_READ_COMMITTED   = 2 ;  
- int TRANSACTION_REPEATABLE_READ  = 4;  
- int TRANSACTION_SERIALIZABLE     = 8;  
- void setTransactionIsolation(int level) throws SQLException;//设置隔离级别，需要在begin之前执行  
- int getTransactionIsolation() throws SQLException; 


- 超时   
**Jdbc对应的api(事务超时)**  
-- 只有Statement有一个设置sql执行超时时间的api   
-- java.sql.Statement.java:   
	 void setQueryTimeout(int seconds) throws SQLException;//名字取得不好，会让人误以为只是设置查询超时，实际update也起作用

- 只读  
**Jdbc对应的api(事务只读)**  
-- java.sql.Connection.java:  
-- void setReadOnly(boolean readOnly) throws SQLException;//设置是否只读，需要在begin之前执行  
-- boolean isReadOnly() throws SQLException;  



## Spring事务框架 ##

![](http://img.blog.csdn.net/20160324011156424)


&emsp;- 常用的DataSourceTransactionManager 
 
    javax.sql.DataSource.java   -> mysql //对应各种数据库  
    Connection getConnection(String username, String password)throws SQLException;  
    
    org.springframework.transaction.PlatformTransactionManager.java -> 事务管理器  
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;//开启一个事务
    void commit(TransactionStatus status) throws TransactionException; //提交一个事务
    void rollback(TransactionStatus status) throws TransactionException;//回滚一个事务
    
    org.springframework.transaction.SavepointManager.java -> 保存点管理器Object 
    createSavepoint() throws TransactionException; //创建保存点
    void rollbackToSavepoint(Object savepoint) throws TransactionException;//回滚保存点void 
    releaseSavepoint(Object savepoint) throws TransactionException;//删除保存点
    TransactionStatus extends SavepointManager
    
    拿到了TransactionStatus后怎么拿到对应Connection执行sql？
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transactionStatus.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();


&emsp;- Spring事务管理器配置   

	<!-- 事务管理器 xml配置-->
	<bean id ='transactionManager'
		class='org.springframework.jdbc.datasource.DataSourceTransactionManager'>
		<property name ='dataSource' ref ='dataSource' />
		<property name ='validateExistingTransaction' value ='true' />
	</bean>

	<tx:annotation-driven transaction-manager ='transactionManager' proxy-target-class='true' />

	@Transactional(readOnly =false, rollbackFor = {Exception.class},noRollbackFor ={Error.class},isolation = Isolation.DEFAULT,propagatio =Propagation.REQUIRED,timeout =5)


&emsp;- spring事务的五大配置参数  
&emsp;&emsp;- 是否只读   
&emsp;&emsp;- 事务的回滚规则(抛出哪些异常时需要回滚，哪些不需要)   
&emsp;&emsp;- 事务的隔离性  
&emsp;&emsp;- 事务的传播行为  
&emsp;&emsp;- 事务的超时  

事务的回滚规则的实现:   
 
具体实现位于org.springframework.transaction.interceptor. RuleBasedTransactionAttribute.java类中的public boolean rollbackOn(Throwable ex);  
- 先遍历rollbackFor，找到一个最接近的配置  
- 再遍历noRollbackFor ，找到一个最接近的配置  
- 如果上面两步找到了一个最接近的配置，属于rollbackFor则回滚，属于noRollbackFor 则不回滚  
- 如果上面两步都没找到一个最接近的配置，则：
- 如果是RuntimeException或者Error的子类就回滚，否则不回滚

事务的传播行为<table>
        <tr>
            <th>类型</th>
            <th>说明</th>
        </tr>
		<tr>
            <th>Propagation.REQUIRED</th>
            <th>代表当前方法支持当前的事务，且与调用者处于同一事务上下文中，回滚统一回滚（如果当前方法是被其他方法调用的时候，且调用者本身即有事务），如果没有事务，则自己新建事务，</th>
        </tr>
		<tr>
            <th>Propagation.SUPPORTS</th>
            <th>代表当前方法支持当前的事务，且与调用者处于同一事务上下文中，回滚统一回滚（如果当前方法是被其他方法调用的时候，且调用者本身即有事务），如果没有事务，则该方法在非事务的上下文中执行</th>
        </tr>
		<tr>
            <th>Propagation.MANDATORY</th>
            <th>代表当前方法支持当前的事务，且与调用者处于同一事务上下文中，回滚统一回滚（如果当前方法是被其他方法调用的时候，且调用者本身即有事务）,如果没有事务，则抛出异常</th>
        </tr>
		<tr>
            <th>Propagation.REQUIRES_NEW</th>
            <th>创建一个新的事务上下文，如果当前方法的调用者已经有了事务，则挂起调用者的事务，这两个事务不处于同一上下文，如果各自发生异常，各自回滚</th>
        </tr>
		<tr>
            <th>Propagation.NOT_SUPPORTED</th>
            <th>该方法以非事务的状态执行，如果调用该方法的调用者有事务则先挂起调用者的事务</th>
        </tr>
		<tr>
            <th>Propagation.NEVER</th>
            <th>该方法以非事务的状态执行，如果调用者存在事务，则抛出异常</th>
        </tr>
		<tr>
            <th>Propagation.NESTED</th>
            <th>如果当前上下文中存在事务，则以嵌套事务执行该方法，也就说，这部分方法是外部方法的一部分，调用者回滚，则该方法回滚，但如果该方法自己发生异常，则自己回滚，不会影响外部事务，如果不存在事务，则与PROPAGATION_REQUIRED一样，（其实这是数据库带有保存点的事务的典型体现，举例来说：旅游行业来说，游客从上海飞巴厘岛，需要从香港进行转机，那么上海~香港就是一个方法且是一个事务，香港~巴厘岛就是嵌入的方法，如果香港~巴厘岛航班取消了，无需回滚上海~香港的，这样代价太大，因为游客已经到了香港，只需修改香港到巴厘岛的航班就可以了，如果上海~香港的飞机取消了，则需要全部事务回滚，其实香港到巴厘岛的航班没有问题</th>
        </tr>
</table>
	
传播行为实现的大致逻辑  
传播行为的实现逻辑都差不多，第一步都是判断当前是否存在事务。那么，如何判断当前是否存在事务？  
通过传递函数参数？   
维护一个全局变量Map，key为当前线程，value不等于null表示已有事务，事务提交或回滚时清除value。如何防止其他线程clear map？ ==》 ThreadLocal   
	
	
事务的传播行为

![](images/spring_transaction.png)
上图中如果虚线对应的事务发生异常，则不回滚Connection所在的事务；如果实线对应的事务发生异常，则回滚Connection所在的事务。

**PROPAGATION_REQUIRES_NEW和PROPAGATION_NESTED**

区别和各自使用场景  
- 本质区别：PROPAGATION_REQUIRES_NEW是新开事务，PROPAGATION_NESTED是在当前事务上创建保存点。  
- 回滚区别：PROPAGATION_REQUIRES_NEW回滚自身事务，不会回滚上层事务；PROPAGATION_NESTED只是回滚到保存点，不会回滚上层事务。  
- 提交区别：PROPAGATION_REQUIRES_NEW提交自身事务，即使后来上层事务回滚，也不会再回滚自身事务；PROPAGATION_NESTED提交只是删除保存点，只能跟随上层事务一起提交，如果后来上层回滚，也跟着回滚。  
 
PROPAGATION_REQUIRES_NEW是自立门户，上层事务可以容忍自身失败的情况下NEW事务提交成功，但是不容忍NEW事务失败的情况下继续执行上层事务。  
类似的场景：  
单独的表用来生成递增的Id，即使Id不连续也没关系，但是上层事务依赖于这个id。  

PROPAGATION_NESTED是没有自立能力的，依附于上层事务，上层事务对它也漠不关心，成功最好，不成功也算了，后续可以根据其他信息推算出来。  
类似的场景：  
比如买东西的时候有小票和发票，打印发票属于内嵌事务，打不了发票你照样可以付钱买东西，小票也可以保修。  

**事务的超时**   
如何检测一个函数是否超时:
定时触发：   
	框架开个定时器(另开线程)，时间到了触发回调函数，回调函数检查目标函数是否执行完毕。
优点是框架时间控制比较精确。  
缺点是如果超时了，该如何提前终止目标函数？

轮询:  
目标函数每次在进入阻塞操作或者耗时操作前轮询一下框架自己是否已经超时。  
优点是开销小、目标函数可以决定在合适的时间和节点提前结束自己的运行。  
缺点是框架无法精确控制超时，全靠目标函数自觉。  

*spring采取的是 轮询检测。
因此如果@Transactional(timeout = 5) 目标函数在执行时不去轮询自己是否超时，即使过了5秒，还是不会超时的。*


**@Transactional冲突**  
是在事务doBegin的时候设置超时的deadline以及隔离级别、设置是否只读（确切的说隔离级别和是否只读是在创建connection后、begin之前的时候设置的，因为这2个底层是由connection设置的）


**@Transactional冲突** 

结论：为了一劳永逸，请用在实现类上。  
annotation加在类上，则jdk代理和cglib代理都可以扫描到。  
annotation加在接口上，只能在jdk代理的时候扫描到。  
为什么加在接口上cglib检测不到  
JDK的动态代理是实现接口方式生成代理类。  
CGLIB动态代理是通过继承方式生成代理类,底层使用ASM框架生成字节码完成代理功能。  
Spring AOP默认首先使用JDK动态代理来代理目标对象，如果目标对象没有实现任何接口将使用CGLIB代理，如果需要强制使用CGLIB代理，可以指定
<aop:aspectj-autoproxy proxy-target-class="true" />


**@Transactional的位置**
可以放在private、final、 protected方法之上吗？   
jdk是代理接口，非public方法必然不会存在在接口里，所以就不会被拦截到，也无法在接口里定义final方法；    
cglib是子类，private的方法照样不会出现在子类里，也不能被拦截。 而且final方法也不能覆盖。   

private、protected如果用到了，则是代码有问题。   
final在jdk代理下有用，cglib下没用。   

## 问题答案 ##
问题1：会。   
问题2：不会。   
问题3：不会。   
问题4：不会。   
问题5：会。   
问题6：不会。   
问题7：超过了数据库innodb_lock_wait_timeout行锁时间。   
问题8：为了cglib代理能识别，都加在实现类的方法上。  
问题9：没用且不合理。   

