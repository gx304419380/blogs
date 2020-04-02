##0x0 现象
系统环境：
>springboot + mybatis + druid + postgresql + activeMQ


下午5点44分，组件阻塞，日志不断报异常：
```
org.springframework.transaction.CannotCreateTransactionException: Could not open JDBC Connection for transaction; 
nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 60000, active 40, maxActive 40, creating 0
	at org.springframework.jdbc.datasource.DataSourceTransactionManager.doBegin(DataSourceTransactionManager.java:289)
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:377)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.createTransactionIfNecessary(TransactionAspectSupport.java:461)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:277)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
    ...
```

可以看到，数据库活跃连接数为40，以达到最大值，尝试获取连接60s没有成功获取到！

##0x1 排查
1. 查看日志，发现报错信息；
2.  进入数据库查看当前连接及正在执行的sql：
>select * from pg_stat_activity;
发现有几十个插入语句事务一直没有结束（事务始终处于开启状态，没有提交）

3. 查看数据库日志：
```
2019-02-16 18:43:27 HKT [73708] 10.66.174.60(52950) pes_pesdb pes_pesdb_user FATAL:  terminating connection due to idle-in-transaction timeout
2019-02-16 18:43:27 HKT [73708] 10.66.174.60(52950) pes_pesdb pes_pesdb_user LOG:  disconnection: session time: 1:01:08.089 user=pes_pesdb_user database=pes_pesdb host=10.66.174.60 port=52950
2019-02-16 18:43:33 HKT [12972] 10.66.174.60(51381) pes_pesdb pes_pesdb_user FATAL:  terminating connection due to idle-in-transaction timeout
2019-02-16 18:43:33 HKT [12972] 10.66.174.60(51381) pes_pesdb pes_pesdb_user LOG:  disconnection: session time: 1:02:13.416 user=pes_pesdb_user 
```
数据库事务超时，被pg强制关闭！
这就导致了一个严重的问题：**数据库连接被数据库端强制关闭了，但是我们druid连接池不知道，所以后来那些阻塞事务都被强制终结后依然获取不到连接！**

4. 查看对应的阻塞sql，发现了问题的原因：
```
伪代码：
//开启事务
begin transaction;
//执行插入语句
insert();
//同步发送mq消息
MqUtils.sendMessage("insert sucess");
//提交事务
commit();
```
这里，**如果发送消息阻塞了，会导致这个事务一直无法提交，从而导致连接被占用，无法释放！**

至此终结问题。
##0xFF 总结
下次出现这个问题，八成就是有某些事务开启后，逻辑上阻塞了，无法关闭导致的。要注意，发送消息这些操作异步执行！

