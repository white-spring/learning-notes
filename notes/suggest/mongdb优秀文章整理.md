# 调度中心技术文章精选

  * [分布式定时任务调度系统技术选型][1]
    * java有哪些定时任务的框架
      * 单机
        * Timer：是一个定时器类，通过该类可以为指定的定时任务进行配置。TimerTask类是一个定时任务类，该类实现了Runnable接口，缺点异常未检查会中止线程
        * ScheduledExecutorService：相对延迟或者周期作为定时任务调度，缺点没有绝对的日期或者时间
        * Spring定时框架：配置简单功能较多，如果系统使用单机的话可以优先考虑spring定时器
      * 分布式
        * Quartz：Java事实上的定时任务标准。但Quartz关注点在于定时任务而非数据，并无一套根据数据处理而定制化的流程。虽然Quartz可以基于数据库实现作业的高可用，但缺少分布式并行调度的功能，不支持任务分片。
        * TBSchedule：阿里早期开源的分布式任务调度系统。代码略陈旧，使用timer而非线程池执行任务调度。众所周知，timer在处理异常状况时是有缺陷的。而且TBSchedule作业类型较为单一，只能是获取/处理数据一种模式。还有就是文档缺失比较严重  
        * elastic-job：当当开发的弹性分布式任务调度系统，功能丰富强大，采用zookeeper实现分布式协调，实现任务高可用以及分片，目前是版本2.15，并且可以支持云开发
        * Saturn：是唯品会自主研发的分布式的定时任务的调度平台，基于当当的elastic-job 版本1开发，并且可以很好的部署到docker容器上。
        * xxl-job: 是大众点评员工徐雪里于2015年发布的分布式任务调度平台，是一个轻量级分布式任务调度框架，其核心设计目标是开发迅速、学习简单、轻量级、易扩展。
 
  * [分布式定时任务对比][2]
  
  |feature          |依赖                  |HA     |任务分片|文档完善|管理界面|难易程度|公司   |高级功能|缺点    |
  |:---------------:|:-------------------:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
  |quartz           |mysql|多节点部署，通过竞争 数据库锁来保证 只有一个节点执行任务| — |完善|无|简单|Open Symphony| — |没有管理界面， 以及不支持任务分片等。 不适用于分布式场景|
  |elastic-job-cloud|jdk1.7+,  zookeeper 3.4.6+, maven3.0.4+, mesos|通过zookeeper的 注册与发现，可以 动态的添加服务器。 支持水平扩容|支持|完善|支持|较复杂|当当网|弹性扩容， 多种作业模式， 失效转移， 运行状态收集， 多线程处理数据， 幂等性， 容错处理， Spring命名空间支持|需要引入zookeeper， mesos, 增加系统复杂度, 学习成本较高|
  |xxl-job          |mysql, jdk1.7+, maven3.0+|集群部署|支持|完善|支持|简单|个人|弹性扩容， 分片广播， 故障转移， Rolling实时日志， GLUE（支持在线 编辑代码，免发布）, 任务进度监控， 任务依赖， 数据加密， 邮件报警， 运行报表， 国际化|调度中心通过获取 DB锁来保证集群中 执行任务的唯一性， 如果短任务很多， 随着调度中心集群 数量增加， 那么数据库的锁竞争 会比较厉害， 性能不好。|
  |antares          |jdk 1.7+, redis, zookeeper|集群部署|支持|文档略少|支持|一般|个人|任务分片， 失效转移， 弹性扩容|不支持动态添加任务|
  |opencron         |jdk1.7+, Tomcat8.0+| — | — |文档略少|支持|一般|个人|时间规则支持 quartz和crontab， kill任务， 现场执行， 查询任务运行状态|不适用于分布式场景|

 
 * [四种分布式任务调度框架对比(quartz,xxl-job,Elastic-Job,Saturn)][3]
   
   * quartz的分布式只是解决了高可用的问题，并没有解决任务分片的问题，还是会有单机处理的极限。
   
   * Saturn， 基于当当Elastic Job代码基础上自主研发的任务调度系统，是唯品会开源的分布式作业调度平台，取代传统的Linux Cron/Spring Batch Job的方式，做到统一配置，统一监控，任务高可用以及分片并发处理。主要是去中心化，高可用，可分片，动态扩容，有认证和授权功能。
 
 * xxl-job 实现分片
   
    * 获取总分片数和当前分片代码：
     
      ```
          
          //获取分片 根据配置的机器数量和获得的分片拿去对应的数据
          
          ShardingUtil.ShardingVO shardingVO = ShardingUtil.getShardingVo();
          
          //执行器数量
          
          int number = shardingVO.getTotal();
          
          //当前分片
          
          int index = shardingVO.getIndex();
      
      ```
    * sql 每次从表中取100条数据：
       
      ```
          SELECT id,name,password
          
          FROM t_push
          
          WHERE `status` = 0
          
          AND mod(id,#{number}) = #{index}  //number 分片总数，index当前分片数
          
          order by id desc
          
          LIMIT 100;
          
      ```
        
 
 
 [1]: https://www.cnblogs.com/davidwang456/p/9057839.html
 [2]: https://blog.csdn.net/u012394095/article/details/79470904
 [3]: https://blog.csdn.net/thver/article/details/87480971