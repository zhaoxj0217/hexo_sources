---

title: Mysql 执行优化—mysql 
data: 2017-11-05
categories: mysql
tags: [mysql]

---

#	Mysql 执行优化
##	认识数据索引
1). 为什么使用数据索引能提高效率  
&emsp;•	数据索引的存储是有序的  
&emsp;•	在有序的情况下，通过索引查询一个数据是无需遍历索引记录的  
&emsp;•	极端情况下，数据索引的查询效率为二分法查询效率，趋近于 log2(N)   
2). 如何理解数据索引的结构   
&emsp;•	数据索引通常默认采用 btree 索引，（内存表也使用了 hash 索引）。  
&emsp;•	单一有序排序序列是查找效率最高的（二分查找，或者说折半查找），使用树形索引的目的是为了达到快速的更新和增删操作。  
&emsp;•	在极端情况下（比如数据查询需求量非常大，而数据更新需求极少，实时性要求不高，数据规模有限），直接使用单一排序序列，折半查找速度最快。  
&emsp;&emsp;&emsp;	实战范例 ： ip 地址反查  
&emsp;&emsp;&emsp;资源： Ip 地址对应表，源数据格式为  startip, endip, area 源数据条数为 10 万条左右，呈很大的分散性目标： 需要通过任意 ip 查询该 ip 所属地区
性能要求达到每秒 1000 次以上的查询效率
&emsp;&emsp;&emsp;挑战： 如使用 between … and 数据库操作，无法有效使用索引。
如果每次查询请求需要遍历 10 万条记录，根本不行。  
&emsp;&emsp;&emsp;方法： 一次性排序（只在数据准备中进行，数据可存储在内存序列）折半查找（每次请求以折半查找方式进行）  
&emsp;•		在进行索引分析和 SQL 优化时，可以将数据索引字段想象为单一有序序列，并以此作为分析的基础。  
&emsp;&emsp;&emsp;	实战范例：复合索引查询优化实战，同城异性列表  
资源： 用户表 user，字段 sex 性别；area 地区；lastlogin 最后登录时间；其他略目标： 查找同一地区的异性，按照最后登录时间逆序    高访问量社区的高频查询，如何优化。  
&emsp;&emsp;&emsp;&emsp;查询 SQL: select * from user where area=’$area’ and sex=’$sex’ order by lastlogin desc limit 0,30;  
&emsp;&emsp;&emsp;&emsp;挑战： 建立复合索引并不难， area+sex+lastlogin 三个字段的复合索引,如何理解？  
&emsp;&emsp;&emsp;&emsp;首先，忘掉 btree，将索引字段理解为一个排序序列。  
&emsp;&emsp;&emsp;&emsp;如果只使用 area 会怎样？搜索会把符合 area 的结果全部找出来，然后在这里面遍历，选择命中 sex 的并排序。 遍历所有 area=’$area’数据！  
如果使用了 area+sex，略好，仍然要遍历所有 area=’$area’ 	and sex=’$sex’数据，然后在这个基础上排序！！  
&emsp;&emsp;&emsp;&emsp;Area+sex+lastlogin 复合索引时（切记 lastlogin 在最后），该索引基于 area+sex+lastlogin 三个字段合并的结果排序，该列表可以想象如下。  
&emsp;&emsp;&emsp;&emsp;广州女$时间 1 广州女$时间 2 广州女$时间 3    
&emsp;&emsp;&emsp;&emsp;…  
&emsp;&emsp;&emsp;&emsp;广州男   
&emsp;&emsp;&emsp;&emsp;….   
&emsp;&emsp;&emsp;&emsp;深圳女  
&emsp;&emsp;&emsp;&emsp;….  
&emsp;&emsp;&emsp;&emsp;数据库很容易命中到 	area+sex 的边界，并且基于下边界向上追溯 30 条记录，搞定！在索引中迅速命中所有结果，无需二次遍历！ 
 
3). 如何理解影响结果集   
&emsp;•	影响结果集是数据查询优化的一个重要中间数据  
&emsp;&emsp;&emsp;	查询条件与索引的关系决定影响结果集如上例所示，即便查询用到了索引，但是如果查询和排序目标不能直接在索引中命中，其可能带来较多的影响结果。而这会直接影响到查询效率  
&emsp;&emsp;&emsp;	微秒级优化  
&emsp;&emsp;&emsp;&emsp;&emsp;•	优化查询不能只看慢查询日志，常规来说，0.01 秒以上的查询，都是不够优化的。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	实战范例  
&emsp;&emsp;&emsp;&emsp;&emsp;和上案例类似，某游戏社区要显示用户动态，select * from userfeed where uid=$uid order by lastlogin desc limit 0,30;   初期默认以 uid 为索引字段，查询为命中所有 uid=$uid 的结果按照 lastlogin 排序。 当用户行为非常频繁时，该 SQL 索引命中影响结果集有数百乃至数千条记录。查询效率超过 0.01 秒，并发较大时数据库压力较大。   
&emsp;&emsp;&emsp;&emsp;&emsp;解决方案：将索引改为 uid+lastlogin 复合索引，索引直接命中影响结果集 30 条，查询效率提高了 10 倍，平均在 0.001 秒，数据库压力骤降。  
&emsp;&emsp;&emsp;	影响结果集的常见误区  
&emsp;&emsp;&emsp;&emsp;&emsp;•	影响结果集并不是说数据查询出来的结果数或操作影响的结果数，而是查询条件的索引所命中的结果数。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	实战范例  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		某游戏数据库使用了 innodb，innodb 是行级锁，理论上很少存在锁表情况。出现了一个 SQL 语句(delete from tabname where xid=…)，这个 SQL 非常用 SQL，仅在特定情况下出现，每天出现频繁度不高（一天仅 10 次左右），数据表容量百万级，但是这个 xid 未建立索引，于是悲惨的事情发生了，当执行这条 delete 的时候，真正删除的记录非常少，也许一到两条，也许一条都没有；但是！由于这个 xid 未建立索引，
delete 操作时遍历全表记录，全表被 delete 操作锁定，select 操作全部被 locked，由于百万条记录遍历时间较长，期间大量 select 被阻塞，数据库连接过多崩溃。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;这种非高发请求，操作目标很少的 SQL，因未使用索引，连带导致整个数据库的查询阻塞，需要极大提高警觉。  
&emsp;&emsp;&emsp;	总结：  
&emsp;&emsp;&emsp;&emsp;&emsp;•	影响结果集是搜索条件索引命中的结果集，而非输出和操作的结果集。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	影响结果集越趋近于实际输出或操作的目标结果集，索引效率越高。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	请注意，我这里永远不会讲关于外键和 join 的优化，因为在我们的体系里，这是根本不允许的！ 架构优化部分会解释为什么。  

##	理解执行状态  
**常见分析手段** 
 
&emsp;•	慢查询日志，关注重点如下  
&emsp;&emsp;&emsp;	是否锁定，及锁定时间  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如存在锁定，则该慢查询通常是因锁定因素导致，本身无需优化，需解决锁定问题。  
&emsp;&emsp;&emsp;	影响结果集  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如影响结果集较大，显然是索引项命中存在问题，需要认真对待。  

&emsp;•	Explain 操作   
&emsp;&emsp;&emsp;	索引项使用  
&emsp;&emsp;&emsp;&emsp;&emsp;•	不建议用 using index 做强制索引，如未如预期使用索引，建议重新斟酌表结构和索引设置。  
&emsp;&emsp;&emsp;	影响结果集  
&emsp;&emsp;&emsp;&emsp;&emsp;•	这里显示的数字不一定准确，结合之前提到对数据索引的理解来看，还记得嘛？就把索引当作有序序列来理解，反思 SQL。
  
&emsp;•	Set profiling , show profiles for query 操作  
&emsp;&emsp;&emsp;执行开销   
&emsp;&emsp;&emsp;&emsp;&emsp;•	注意，有问题的 SQL 如果重复执行，可能在缓存里，这时要注意避免缓存影响。通过这里可以看到。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	执行时间超过 0.005 秒的频繁操作 SQL 建议都分析一下。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	深入理解数据库执行的过程和开销的分布  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		Show processlist   
&emsp;•状态清单  
&emsp;&emsp;&emsp;Sleep 状态， 通常代表资源未释放，如果是通过连接池，sleep 状态应该恒定在一定数量范围内   
&emsp;&emsp;&emsp;&emsp;&emsp;•	实战范例： 因前端数据输出时（特别是输出到用户终端）未及时关闭数据库连接，导致因网络连接速度产生大量 sleep 连接，在网速  
出现异常时，数据库 too many connections 挂死。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	简单解读，数据查询和执行通常只需要不到 0.01 秒，而网络输出通常需要 1 秒左右甚至更长，原本数据连接在 0.01 秒即可释放，但是因为前端程序未执行 close 操作，直接输出结果，那么在结果未展现在用户桌面前，该数据库连接一直维持在 sleep 状态！  
&emsp;&emsp;&emsp;	Waiting for net, reading from net, writing to net  
&emsp;&emsp;&emsp;&emsp;&emsp;•	偶尔出现无妨  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如大量出现，迅速检查数据库到前端的网络连接状态和流量  
&emsp;&emsp;&emsp;&emsp;&emsp;•	案例: 因外挂程序，内网数据库大量读取，内网使用的百兆交换迅速爆满，导致大量连接阻塞在 waiting for net，数据库连接过多崩溃  
&emsp;&emsp;&emsp;	Locked 状态  
&emsp;&emsp;&emsp;&emsp;&emsp;•	有更新操作锁定  
&emsp;&emsp;&emsp;&emsp;&emsp;•	通常使用 innodb 可以很好的减少 locked 状态的产生，但是切记，更新操作要正确使用索引，即便是低频次更新操作也不能疏忽。如上影响结果集范例所示。   
&emsp;&emsp;&emsp;&emsp;&emsp;•	在 myisam 的时代，locked 是很多高并发应用的噩梦。所以 mysql 官方也开始倾向于推荐 innodb。  
&emsp;&emsp;&emsp;	Copy to tmp table  
&emsp;&emsp;&emsp;&emsp;&emsp;•	索引及现有结构无法涵盖查询条件，才会建立一个临时表来满足查询要求，产生巨大的恐怖的 i/o 压力。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	很可怕的搜索语句会导致这样的情况，如果是数据分析，或者半夜的周期数据清理任务，偶尔出现，可以允许。频繁出现务必优化之。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	Copy to tmp table 通常与连表查询有关，建议逐渐习惯不使用连表查询。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	实战范例：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■ 某社区数据库阻塞，求救，经查，其服务器存在多个数据库应用和网站，其中一个不常用的小网站数据库产生了一个恐怖的copy to tmp table 操作，导致整个硬盘 i/o 和 cpu 压力超载。Kill 掉该操作一切恢复。  
&emsp;&emsp;&emsp;	Sending data  
&emsp;&emsp;&emsp;&emsp;&emsp;•	Sending data 并不是发送数据，别被这个名字所欺骗，这是从物理磁盘获取数据的进程，如果你的影响结果集较多，那么就需要从不同的磁盘碎片去抽取数据，  
&emsp;&emsp;&emsp;&emsp;&emsp;•	偶尔出现该状态连接无碍。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	回到上面影响结果集的问题，一般而言，如果 sending data 连接过多，通常是某查询的影响结果集过大，也就是查询的索引项不够优化。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如果出现大量相似的 SQL 语句出现在 show proesslist 列表中，并且都处于 sending data 状态，优化查询索引，记住用影响结果集的思路去思考。  
&emsp;&emsp;&emsp;	Freeing items  
&emsp;&emsp;&emsp;&emsp;&emsp;•	理论上这玩意不会出现很多。偶尔出现无碍  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如果大量出现，内存，硬盘可能已经出现问题。比如硬盘满或损坏。  
&emsp;&emsp;&emsp;	Sorting for …  
&emsp;&emsp;&emsp;&emsp;&emsp;• 和 Sending 	data 类似，结果集过大，排序条件没有索引化，需要在内存里排序，甚至需要创建临时结构排序。  
&emsp;&emsp;&emsp;	其他  
&emsp;&emsp;&emsp;&emsp;&emsp;• 还有很多状态，遇到了，去查查资料。基本上我们遇到其他状态的阻塞较少，所以不关心。

##分析流程  
&emsp;•	基本流程  
&emsp;&emsp;&emsp;	详细了解问题状况   
&emsp;&emsp;&emsp;&emsp;&emsp;•	Too many connections 是常见表象，有很多种原因。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	索引损坏的情况在 innodb 情况下很少出现。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如出现其他情况应追溯日志和错误信息。  
&emsp;&emsp;&emsp;	了解基本负载状况和运营状况  
&emsp;&emsp;&emsp;&emsp;&emsp;•	基本运营状况  
&emsp;•		当前每秒读请求  
&emsp;•		当前每秒写请求  
&emsp;•		当前在线用户  
&emsp;•		当前数据容量 
 
**基本负载情况 **   
&emsp;•	学会使用这些指令  
&emsp;&emsp;&emsp;	Top   
&emsp;&emsp;&emsp;	Vmstat  
&emsp;&emsp;&emsp;	uptime   
&emsp;&emsp;&emsp;	iostat   
&emsp;&emsp;&emsp;	df   
&emsp;•	Cpu 负载构成  
&emsp;&emsp;&emsp;	特别关注 i/o 压力( wa%)  
&emsp;&emsp;&emsp;	多核负载分配  
&emsp;•	内存占用  
&emsp;&emsp;&emsp;	Swap 分区是否被侵占  
&emsp;&emsp;&emsp;	如 Swap 分区被侵占，物理内存是否较多空闲  
&emsp;•	磁盘状态  
&emsp;&emsp;&emsp;	硬盘满和 inode 节点满的情况要迅速定位和迅速处理  
&emsp;&emsp;&emsp;	了解具体连接状况  

**当前连接数**   
&emsp;•	Netstat –an|grep 3306|wc –l  
&emsp;•	Show processlist  

**当前连接分布 show processlist**    
&emsp;•	前端应用请求数据库不要使用 root 帐号！  
&emsp;&emsp;&emsp;	Root 帐号比其他普通帐号多一个连接数许可。  
&emsp;&emsp;&emsp;	前端使用普通帐号，在 too many connections 的时候 root 帐号仍可以登录数据库查询 show processlist!  
&emsp;&emsp;&emsp;	记住，前端应用程序不要设置一个不叫 root 的 root 帐号来糊弄！非 root 账户是骨子里的，而不是名义上的。  
&emsp;•	状态分布  
&emsp;&emsp;&emsp;	不同状态代表不同的问题，有不同的优化目标。  
&emsp;&emsp;&emsp;	参见如上范例。  
&emsp;•	雷同 SQL 的分布  
&emsp;&emsp;&emsp;	是否较多雷同 SQL 出现在同一状态  

**当前是否有较多慢查询日志**  
&emsp;•	是否锁定  
&emsp;•	影响结果集  
&emsp;&emsp;&emsp;	频繁度分析  

**写频繁度**  
&emsp;•	如果 i/o 压力高，优先分析写入频繁度  
&emsp;•	Mysqlbinlog 输出最新 binlog 文件，编写脚本拆分  
&emsp;•	最多写入的数据表是哪个  
&emsp;•	最多写入的数据 SQL 是什么  
&emsp;•	是否存在基于同一主键的数据内容高频重复写入？  
&emsp;•	涉及架构优化部分，参见架构优化-缓存异步更新  
**读取频繁度**  
&emsp;•	如果 cpu 资源较高，而 i/o 压力不高，优先分析读取频繁度  
&emsp;•	程序中在封装的 db 类增加抽样日志即可，抽样比例酌情考虑，以不显著影响系统负载压力为底线。  
&emsp;•	最多读取的数据表是哪个  
&emsp;•	最多读取的数据 SQL 是什么  
&emsp;&emsp;&emsp;	该 SQL 进行 explain 和 set profiling 判定  
&emsp;&emsp;&emsp;	注意判定时需要避免 query cache 影响  
&emsp;&emsp;&emsp;	比如，在这个 SQL 末尾增加一个条件子句 and 1=1 就可以
避免从 query cache 中获取数据，而得到真实的执行状态分析。   
&emsp;• 是否存在同一个查询短期内频繁出现的情况  
&emsp;&emsp;&emsp;	涉及前端缓存优化  
&emsp;&emsp;&emsp;	抓大放小，解决显著问题 
 
**不苛求解决所有优化问题，但是应以保证线上服务稳定可靠为目标。**
  
**解决与评估要同时进行，新的策略或解决方案务必经过评估后上线。**
  
##总结  
&emsp;•	要学会怎样分析问题，而不是单纯拍脑袋优化  
&emsp;•	慢查询只是最基础的东西，要学会优化 0.01 秒的查询请求。  
&emsp;•	当发生连接阻塞时，不同状态的阻塞有不同的原因，要找到原因，如果不对症下药，就会南辕北辙  
&emsp;&emsp;&emsp;	范例：如果本身系统内存已经超载，已经使用到了 swap，而还在考虑加大缓存来优化查询，那就是自寻死路了。  
&emsp;•	监测与跟踪要经常做，而不是出问题才做  
&emsp;&emsp;&emsp;	读取频繁度抽样监测  
&emsp;&emsp;&emsp;&emsp;&emsp;•	全监测不要搞，i/o 吓死人。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	按照一个抽样比例抽样即可。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	针对抽样中发现的问题，可以按照特定 SQL 在特定时间内监测一段全查询记录，但仍要考虑 i/o 影响。     
&emsp;&emsp;&emsp;	写入频繁度监测  
&emsp;&emsp;&emsp;&emsp;&emsp;•	基于 binlog 解开即可，可定时或不定时分析。  
&emsp;&emsp;&emsp;	微慢查询抽样监测  
&emsp;&emsp;&emsp;&emsp;&emsp;•	高并发情况下，查询请求时间超过 0.01 秒甚至 0.005 秒的，建议酌情抽样记录。  
&emsp;&emsp;&emsp;	连接数预警监测  
&emsp;&emsp;&emsp;&emsp;&emsp;•	连接数超过特定阈值的情况下，虽然数据库没有崩溃，建议记录相关连接状态。  
&emsp;•	学会通过数据和监控发现问题，分析问题，而后解决问题顺理成章。特别是要学会在日常监控中发现隐患，而不是问题爆发了才去处理和解决。  

