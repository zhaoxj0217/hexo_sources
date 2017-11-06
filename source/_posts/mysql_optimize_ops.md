---

title: Mysql 运维优化—mysql 
data: 2017-11-05
categories: mysql
tags: [mysql]

---

# Mysql 运维优化 
## 存储引擎类型 
&emsp;•	Myisam 速度快，响应快。表级锁是致命问题。  
&emsp;•	Innodb 目前主流存储引擎  
&emsp;&emsp;&emsp;•	行级锁   
&emsp;&emsp;&emsp;&emsp;&emsp;•	务必注意影响结果集的定义是什么  
&emsp;&emsp;&emsp;&emsp;&emsp;•	行级锁会带来更新的额外开销，但是通常情况下是值得的。  
&emsp;&emsp;&emsp;•	事务提交   
&emsp;&emsp;&emsp;&emsp;&emsp;•	对 i/o 效率提升的考虑  
&emsp;&emsp;&emsp;&emsp;&emsp;•	对安全性的考虑  
&emsp;•	HEAP 内存引擎  
&emsp;&emsp;&emsp;•	频繁更新和海量读取情况下仍会存在锁定状况  
## 内存使用考量 
&emsp;•	理论上，内存越大，越多数据读取发生在内存，效率越高  
&emsp;•	要考虑到现实的硬件资源和瓶颈分布  
&emsp;•	学会理解热点数据，并将热点数据尽可能内存化  
&emsp;&emsp;&emsp;•	所谓热点数据，就是最多被访问的数据。  
&emsp;&emsp;&emsp;•	通常数据库访问是不平均的，少数数据被频繁读写，而更多数据鲜有读写。  
&emsp;&emsp;&emsp;•	学会制定不同的热点数据规则，并测算指标。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	热点数据规模，理论上，热点数据越少越好，这样可以更好的满足业务的增长趋势。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	响应满足度，对响应的满足率越高越好。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	比如依据最后更新时间，总访问量，回访次数等指标定义热点数据，并测算不同定义模式下的热点数据规模  
## 性能与安全性考量   
&emsp;•	数据提交方式  
&emsp;&emsp;&emsp;•	innodb_flush_log_at_trx_commit = 1 每次自动提交，安全性高，i/o 压力大  
&emsp;&emsp;&emsp;•	innodb_flush_log_at_trx_commit = 2 每秒自动提交，安全性略有影响，i/o 承载强。  
&emsp;•	日志同步  
&emsp;&emsp;&emsp;•	Sync-binlog	=1 每条自动更新，安全性高，i/o 压力大  
&emsp;&emsp;&emsp;•	Sync-binlog = 0 根据缓存设置情况自动更新，存在丢失数据和同步延迟风险，i/o 承载力强。  
&emsp;•	性能与安全本身存在相悖的情况，需要在业务诉求层面决定取舍
&emsp;&emsp;&emsp;•	学会区分什么场合侧重性能，什么场合侧重安全  
&emsp;&emsp;&emsp;•	学会将不同安全等级的数据库用不同策略管理  
## 存储压力优化   
&emsp;•	顺序读写性能远高于随机读写  
&emsp;•	日志类数据可以使用顺序读写方式进行  
&emsp;•	将顺序写数据和随机读写数据分成不同的物理磁盘，有助于 i/o 压力的疏解，前提是，你确信你的 i/o 压力主要来自于可顺序写操作（因随机读写干扰导致不能顺序写，但是确实可以用顺序写方式进行的 i/o 操作）。  
## 运维监控体系
&emsp;•	系统监控  
&emsp;&emsp;&emsp;•	服务器资源监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	Cpu, 内存，硬盘空间，i/o 压力  
&emsp;&emsp;&emsp;&emsp;&emsp;•	设置阈值报警  
&emsp;&emsp;&emsp;•	服务器流量监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	外网流量，内网流量  
&emsp;&emsp;&emsp;&emsp;&emsp;•	设置阈值报警  
&emsp;&emsp;&emsp;•	连接状态监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	Show processlist 设置阈值，每分钟监测，超过阈值记录  
&emsp;•	应用监控  
&emsp;&emsp;&emsp;•	慢查询监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	慢查询日志  
&emsp;&emsp;&emsp;&emsp;&emsp;•	如果存在多台数据库服务器，应有汇总查阅机制。  
&emsp;&emsp;&emsp;•	请求错误监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	高频繁应用中，会出现偶发性数据库连接错误或执行错误，将错误信息记录到日志，查看每日的比例变化。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	偶发性错误，如果数量极少，可以不用处理，但是需时常监控其趋势。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	会存在恶意输入内容，输入边界限定缺乏导致执行出错，需基于此防止恶意入侵探测行为。  
&emsp;&emsp;&emsp;•	微慢查询监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	高并发环境里，超过 0.01 秒的查询请求都应该关注一下。  
&emsp;&emsp;&emsp;•	频繁度监控  
&emsp;&emsp;&emsp;&emsp;&emsp;•	写操作，基于 binlog，定期分析。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	读操作，在前端 db 封装代码中增加抽样日志，并输出执行时间。  
&emsp;&emsp;&emsp;&emsp;&emsp;•	分析请求频繁度是开发架构 进一步优化的基础  
&emsp;&emsp;&emsp;&emsp;&emsp;•	最好的优化就是减少请求次数！
  
## 总结：
&emsp;&emsp;&emsp;•	监控与数据分析是一切优化的基础。  
&emsp;&emsp;&emsp;•	没有运营数据监测就不要妄谈优化！  
监控要注意不要产生太多额外的负载，不要因监控带来太多额外系统开销  
