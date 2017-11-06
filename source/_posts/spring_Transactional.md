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


事务的代码流转  
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
- 隔离级别  
<table>
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
- 只读  








