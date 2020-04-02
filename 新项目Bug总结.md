##0x1 while循环退出条件要进行测试，杜绝死循环
>bug代码：
```
            LocalTime start = task.getExecTimeStart();
            LocalTime end = task.getExecTimeEnd();
            Integer interval = task.getTimeInterval();

            while (!start.isAfter(end)) {
                if (start.isAfter(limitTime)) {
                    execTimeList.add(start);
                }
                //这里如果start是23：55，加10分钟后就是0:05
                //如果end是23:59那么，会导致死循环！
                start = start.plusMinutes(interval);
            }
```
##0x2 避免在循环中进行new对象、尽量使用原始数据类型



##0x3 Map集合获取到的数据要进行判空

##0x4 关于数据库排序问题：
按照日期排序，如果某个日期下对应许多数据，例如2019-10-10下有10000条数据，此时按照日期排序分页查询，数据库排序算法不稳定，会导致分页失效：
```
--如果对应日期下，数据超过20条，可能会导致分页失效......
select * from table order by data_time desc limit 20 offset 0
```
解决方案：排序字段加上id：
```
select * from table order by data_time desc, data_id limit 20 offset 0
```
