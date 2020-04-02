因为spring的事务是通过AOP实现的，因此在service中加锁，锁在事务内部开启的，放锁后，事务可能还没提交；

```
//伪代码
method() {
    beginTransaction();
    try {   
        //你的锁是被代理方法里加的
       targetMethod.invoke();
       commit();
    } catch (e) {
       rollback();   
    }
}
```
**在高并发情况下，这个锁其实是无效的。**
例如：
```
//伪代码
begin transaction;

synchronized {
    //检查name是否存在
    boolean exist = checkExist(name);
    //如果那么已存在，返回
    if (exist) return;
    //如果name不存在，插入数据库
    insert(name);
}
commit;
```
以上的代码，在高并发情况下，例如两个线程A和B同时执行：
>  A线程开启事务，将要插入name="张三"；
B线程开启事务，也要插入name="张三";
A线程获取锁，B线程阻塞等待锁；
A线程检查name="张三"不存在，执行插入数据库操作；
A线程释放锁，B线程拿到锁；
此时B线程查询"张三"也不存在（因为A还未提交事务）；
B线程也插入了一条"张三"；
A提交事务；
B提交事务；

至此，会发现数据库有两个张三！
