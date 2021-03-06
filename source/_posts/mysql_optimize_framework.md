---

title: Mysql 架构优化—mysql 
data: 2017-11-05
categories: mysql
tags: [mysql]

---


#	Mysql 架构优化
##	架构优化目标  
1). 防止单点隐患  
&emsp;•	所谓单点隐患，就是某台设备出现故障，会导致整体系统的不可用，这个设备就是单点隐患。   
&emsp;•	理解连带效应，所谓连带效应，就是一种问题会引发另一种故障，举例而言，
memcache+mysql 是一种常见缓存组合，在前端压力很大时，如果 memcache 崩溃，
理论上数据会通过 mysql 读取，不存在系统不可用情况，但是 mysql 无法对抗如此大的压力冲击，会因此连带崩溃。因 A 系统问题导致 B 系统崩溃的连带问题，在运维过程中会频繁出现。     
&emsp;&emsp;&emsp;	实战范例： 在 mysql 连接不及时释放的应用环境里，当网络环境异常（同机房友邻服务器遭受拒绝服务攻击，出口阻塞），网络延迟加剧，空连接数急剧增加，导致数据库连接过多崩溃。  
&emsp;&emsp;&emsp;	实战范例 2：前端代码 通常我们封装 mysql_connect 和 memcache_connect，二者的顺序不同，会产生不同的连带效应。如果 mysql_connect 在前，那么一旦 memcache 连接阻塞，会连带 mysql 空连接过多崩溃。  
&emsp;&emsp;&emsp;	连带效应是常见的系统崩溃，日常分析崩溃原因的时候需要认真考虑连带效应的影响，头疼医头，脚疼医脚是不行的。  

2). 方便系统扩容  
&emsp;•	数据容量增加后，要考虑能够将数据分布到不同的服务器上。  
&emsp;•	请求压力增加时，要考虑将请求压力分布到不同服务器上。  
&emsp;•	扩容设计时需要考虑防止单点隐患。  

3). 安全可控，成本可控  
&emsp;•	数据安全，业务安全  
&emsp;•	人力资源成本>带宽流量成本>硬件成本  
&emsp;&emsp;&emsp;	成本与流量的关系曲线应低于线性增长（流量为横轴，成本为纵轴）。  
&emsp;&emsp;&emsp;	规模优势  
&emsp;•	本教程仅就与数据库有关部分讨论，与数据库无关部门请自行参阅其他学习资料。  

##分布式方案  
1). 分库&拆表方案  
&emsp;•	基本认识  
&emsp;&emsp;&emsp;	用分库&拆表是解决数据库容量问题的唯一途径。  
&emsp;&emsp;&emsp;	分库&拆表也是解决性能压力的最优选择。  
&emsp;&emsp;&emsp;	分库 – 不同的数据表放到不同的数据库服务器中（也可能是虚拟服务器）  
&emsp;&emsp;&emsp;	拆表 – 一张数据表拆成多张数据表，可能位于同一台服务器，也可能位于多台服务器（含虚拟服务器）。  

&emsp;•	去关联化原则  
&emsp;&emsp;&emsp;	摘除数据表之间的关联，是分库的基础工作。  
&emsp;&emsp;&emsp;	摘除关联的目的是，当数据表分布到不同服务器时，查询请求容易分发和处理。  
&emsp;&emsp;&emsp;	学会理解反范式数据结构设计，所谓反范式，第一要点是不用外键，不允许
Join 操作，不允许任何需要跨越两个表的查询请求。第二要点是适度冗余减少查询请求，比如说，信息表，fromuid, touid, message 字段外，还需要一个 fromuname 字段记录用户名，这样查询者通过 touid 查询后，能够立即得到发信人的用户名，而无需进行另一个数据表的查询。  
&emsp;&emsp;&emsp;	去关联化处理会带来额外的考虑，比如说，某一个数据表内容的修改，对另一个数据表的影响。这一点需要在程序或其他途径去考虑。  

&emsp;•	分库方案  
&emsp;&emsp;&emsp;	安全性拆分  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	将高安全性数据与低安全性数据分库，这样的好处第一是便于维护，第二是高安全性数据的数据库参数配置可以以安全优先，而低安全性数据的参数配置以性能优先。参见运维优化相关部分。  
&emsp;&emsp;&emsp;	顺序写数据与随机读写数据分库  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		顺序数据与随机数据区分存储地址，保证物理 i/o 优化。这个实话说，我只听说了概念，还没学会怎么实践。  
&emsp;&emsp;&emsp;	基于业务逻辑拆分  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	根据数据表的内容构成，业务逻辑拆分，便于日常维护和前端调用。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	基于业务逻辑拆分，可以减少前端应用请求发送到不同数据库服务器的频次，从而减少链接开销。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	基于业务逻辑拆分，可保留部分数据关联，前端 web 工程师可在限度范围内执行关联查询。  
&emsp;&emsp;&emsp;	基于负载压力拆分  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	基于负载压力对数据结构拆分，便于直接将负载分担给不同的服务器。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	基于负载压力拆分，可能拆分后的数据库包含不同业务类型的数据表，日常维护会有一定的烦恼。  

&emsp;•	分表方案  
&emsp;&emsp;&emsp;	数据量过大或者访问压力过大的数据表需要切分  
&emsp;&emsp;&emsp;	忙闲分表  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	单数据表字段过多，可将频繁更新的整数数据与非频繁更新的字符串数据切分  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	范例 user 表 ，个人简介，地址，QQ 号，联系方式，头像 这些字段为字符串类型，更新请求少； 最后登录时间，在线时常，访问次数，信件数这些字段为整数型字段，更新频繁，可以将后面这些更新频繁的字段独立拆出一张数据表，表内容变少，索引结构变少，读写请求变快。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■	横向切表  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;● 等分切表，如哈希切表或其他基于对某数字取余的切表。等分切表的优点是负载很方便的分布到不同服务器；缺点是当容量继续增加时无法方便的扩容，需要重新进行数据的切分或转表。而且一些关键主键不易处理。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	递增切表，比如每 1kw 用户开一个新表，优点是可以适应数据的自增趋势；缺点是往往新数据负载高，压力分配不平均。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	日期切表，适用于日志记录式数据，优缺点等同于递增切表。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	个人倾向于递增切表，具体根据应用场景决定。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		热点数据分表  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	将数据量较大的数据表中将读写频繁的数据抽取出来，形成热点数据表。通常一个庞大数据表经常被读写的内容往往具有一定的集中性，如果这些集中数据单独处理，就会极大减少整体系统的负载。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	热点数据表与旧有数据关系  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	可以是一张冗余表，即该表数据丢失不会妨碍使用，因源数据仍存在于旧有结构中。优点是安全性高，维护方便，缺点是写压力不能分担，仍需要同步写回原系统。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	可以是非冗余表，即热点数据的内容原有结构不再保存，优点是读写效率全部优化；缺点是当热点数据发生变化时，维护量较大。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	具体方案选择需要根据读写比例决定，在读频率远高于写频率情况下，优先考虑冗余表方案。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●		热点数据表可以用单独的优化的硬件存储，比如昂贵的闪存卡或大内存系统。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●		热点数据表的重要指标  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*		热点数据的定义需要根据业务模式自行制定策略，常见策略为，按照最新的操作时间；按照内容丰富度等等。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*		数据规模，比如从 1000 万条数据，抽取出 100 万条热点数据。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*		热点命中率，比如查询 10 次，多少次命中在热点数据内。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*		理论上，数据规模越小，热点命中率越高，说明效果越好。需要根据业务自行评估。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	热点数据表的动态维护  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	加载热点数据方案选择  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	定时从旧有数据结构中按照新的策略获取  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	在从旧有数据结构读取时动态加载到热点数据  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;●	剔除热点数据方案选择  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	基于特定策略，定时将热点数据中访问频次较少的数据剔除  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;*	如热点数据是冗余表，则直接删除即可，如不是冗余表，需要回写给旧有数据结构。  
&emsp;&emsp;通常，热点数据往往是基于缓存或者 key-value 方案冗余存储，所以这里提到的热点数据表，其实更多是理解思路，用到的场合可能并不多….  

&emsp;•	表结构设计  
&emsp;&emsp;&emsp;查询冗余表设计  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		涉及分表操作后，一些常见的索引查询可能需要跨表，带来不必要的麻烦。 确认查询请求远大于写入请求时，应设置便于查询项的冗余表。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		实战范例，  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		用户分表，将用户库分成若干数据表  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		基于用户名的查询和基于 uid 的查询都是高并发请求。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■			用户分表基于 uid 分成数据表，同时基于用户名做对应冗余表。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆冗余表要点  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		数据一致性，简单说，同增，同删，同更新。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		可以做全冗余，或者只做主键关联的冗余，比如通过用户名查询 uid，再基于 uid 查询源表。  
&emsp;&emsp;&emsp;	中间数据表  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		为了减少会涉及大规模影响结果集的表数据操作，比如 count，sum 操作。应将一些统计类数据通过中间数据表保存。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		中间数据表应能通过源数据表恢复。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		实战范例：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		论坛板块的发帖量，回帖量，每日新增数据等  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		网站每日新增用户数等。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		后台可以通过源数据表更新该数字。  
&emsp;&emsp;&emsp;	历史数据表  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		历史数据表对应于热点数据表，将需求较少又不能丢弃的数据存入，仅在少数情况下被访问。  

2). 主从架构  
&emsp;•	基本认识  
&emsp;&emsp;&emsp;	读写分离对负载的减轻远远不如分库分表来的直接。  
&emsp;&emsp;&emsp;	写压力会传递给从表，只读从库一样有写压力，一样会产生读写锁！  
&emsp;&emsp;&emsp;	一主多从结构下，主库是单点隐患，很难解决（如主库当机，从库可以响应读写，但是无法自动担当主库的分发功能）  
&emsp;&emsp;&emsp;	主从延迟也是重大问题。一旦有较大写入问题，如表结构更新，主从会产生巨大延迟。  

&emsp;•	应用场景  
&emsp;&emsp;&emsp;	在线热备  
&emsp;&emsp;&emsp;	异地分布  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	写分布，读统一。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	仍然困难重重，受限于网络环境问题巨多！  
&emsp;&emsp;&emsp;	自动障碍转移  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	主崩溃，从自动接管  
&emsp;&emsp;&emsp;	个人建议，负载均衡主要使用分库方案，主从主要用于热备和障碍转移。 
 
&emsp;•	潜在优化点  
&emsp;&emsp;&emsp;	为了减少写压力，有些人建议主不建索引提升 i/o 性能，从建立索引满足查询要求。个人认为这样维护较为麻烦。而且从本身会继承主的 i/o 压力，因此优化价值有限。该思路特此分享，不做推荐。  

3). 故障转移处理  
&emsp;•		要点  
&emsp;&emsp;&emsp;	程序与数据库的连接，基于虚地址而非真实 ip，由负载均衡系统监控。  
&emsp;&emsp;&emsp;	保持主从结构的简单化，否则很难做到故障点摘除。  

&emsp;•		思考方式  
&emsp;&emsp;&emsp;	遍历对服务器集群的任何一台服务器，前端 web，中间件，监控，缓存，db 等等，假设该服务器出现故障，系统是否会出现异常？用户访问是否会出现异常。  
&emsp;&emsp;&emsp;	目标：任意一台服务器崩溃，负载和数据操作均会很短时间内自动转移到其他服务器，不会影响业务的正常进行。不会造成恶性的数据丢失。（哪些是可以丢失的，哪些是不能丢失的）  

##	缓存方案  
1). 缓存结合数据库的读取  
&emsp;•		Memcached/redis 是最常用的缓存系统  
&emsp;•		Mysql 最新版本已经开始支持 memcache 插件，但据牛人分析，尚不成熟，暂不推荐。  
&emsp;•		数据读取  
&emsp;&emsp;&emsp;	并不是所有数据都适合被缓存，也并不是进入了缓存就意味着效率提升。  
&emsp;&emsp;&emsp;	命中率是第一要评估的数据。  
&emsp;&emsp;&emsp;	如何评估进入缓存的数据规模，以及命中率优化，是非常需要细心分析的。  
&emsp;•	实景分析： 前端请求先连接缓存，缓存未命中连接数据库，进行查询，未命中状态比单纯连接数据库查询多了一次连接和查询的操作；如果缓存命中率很低，则这个额外的操作非但不能提高查询效率，反而为系统带来了额外的负载和复杂性，得不偿失。  
&emsp;&emsp;&emsp;	相关评估类似于热点数据表的介绍。  
&emsp;&emsp;&emsp;	善于利用内存，请注意数据存储的格式及压缩算法。  
&emsp;•	Key-value 方案繁多，本培训文档暂不展开。  

2). 缓存结合数据库的写入  
&emsp;•	利用缓存不但可以减少数据读取请求，还可以减少数据库写入 i/o 压力  
&emsp;•	缓存实时更新，数据库异步更新  
&emsp;&emsp;&emsp;	缓存实时更新数据，并将更新记录写入队列  
&emsp;&emsp;&emsp;	可以使用类似 mq 的队列产品，自行建立队列请注意使用 increment 来维持队列序号。  
&emsp;&emsp;&emsp;	不建议使用 get 后处理数据再 set 的方式维护队列  
&emsp;•	测试范例：  
&emsp;•	范例 1
   
    $var=Memcache_get($memcon,”var”);  
     $var++;  
    memcache_set($memcon,”var”,$var);  

&emsp;这样一个脚本，使用 apache ab 去跑，100 个并发，跑 10000 次，然后输出缓存存取的数据，很遗憾，并不是 1000，而是 5000 多，6000 多这样的数字，中间的数字全在 get & set 的过程中丢掉了。  
原因，读写间隔中其他并发写入，导致数据丢失。  
&emsp;•	范例 2  
&emsp;用 memcache_increment 来做这个操作，同样跑测试会得到完整的 10000，一条数据不会丢。  
&emsp;•	结论： 用 increment 存储队列编号，用标记+编号作为 key 存储队列内容。  
&emsp;&emsp;&emsp;	后台基于缓存队列读取更新数据并更新数据库  
&emsp;•	基于队列读取后可以合并更新  
&emsp;•	更新合并率是重要指标  
&emsp;•	实战范例：  
某论坛热门贴，前端不断有 views=views+1 数据更新请求。  
缓存实时更新该状态后台任务对数据库做异步更新时，假设执行周期是 5 分钟，那么五分钟可能
会接收到这样的请求多达数十次乃至数百次，合并更新后只执行一次 update 即可。
类似操作还包括游戏打怪，生命和经验的变化；个人主页访问次数的变化等。  
&emsp;&emsp;&emsp;	异步更新风险  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	前后端同时写，可能导致覆盖风险。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	使用后端异步更新，则前端应用程序就不要写数据库，否则可能造成写入冲突。一种兼容的解决方案是，前端和后端不要写相同的字段。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆	实战范例：  
&emsp;&emsp;&emsp;&emsp;&emsp;用户在线上时，后台异步更新用户状态。管理员后台屏蔽用户是直接更新数据库。  
结果管理员屏蔽某用户操作完成后，因该用户在线有操作，后台异步更新程序再次基于缓存更新用户状态，用户状态被复活，屏蔽失效。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		缓存数据丢失或服务崩溃可能导致数据丢失风险。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		如缓存中间出现故障，则缓存队列数据不会回写到数据库，而用户会认为已经完成，此时会带来比较明显的用户体验问题。  
&emsp;&emsp;&emsp;&emsp;&emsp;◆		一个不彻底的解决方案是，确保高安全性，高重要性数据实时数据更新，而低安全性数据通过缓存异步回写方式完成。此外，使用相对数值操作而不是绝对数值操作更安全。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		范例：支付信息，道具的购买与获得，一旦丢失会对用户造成极大的伤害。而经验值，访问数字，如果只丢失了很少时间的内容，用户还是可以容忍的。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;■		范例：如果使用 Views=Views+…的操作，一旦出现数据格式错误，从 binlog 中反推是可以进行数据还原，但是如果使用 Views=特定值的操作，一旦缓存中数据有错误，则直接被赋予了一个错误数据，无法回溯！  
&emsp;&emsp;&emsp;	异步更新如出现队列阻塞可能导致数据丢失风险。  
&emsp;&emsp;&emsp;	异步更新通常是使用缓存队列后，在后台由 cron 或其他守护进程写入数据库。  
如果队列生成的速度>后台更新写入数据库的速度，就会产生阻塞，导致数据越累计越多，数据库响应迟缓，而缓存队列无法迅速执行，导致溢出或者过期失效。  
