0x0 背景
==
有时候我们单元测试使用的数据库可能是本地数据库，如果在其他地方运行单元测试，往往由于数据库不一致而失败，因此使用一个临时创建的内存行数据库成为一个好的选择，本文使用H2内存行数据库。

0x1 方法
==
1）引用依赖(gradle语法)
```
testCompile 'com.h2database:h2'
```

2）springboot配置
```
spring:
    datasource:
       type: com.alibaba.druid.pool.DruidDataSource
       #H2内存数据库，测试使用数据库
       username:          #空（即" "，可能不写也行）
       password:          #空
       schema: classpath:initSchema.sql  #这里填写你要初始化表的脚本
       data: classpath:initData.sql      #这里填写你要加载的数据的脚本
       druid:
          url: jdbc:h2:mem:test;mode=PostgreSQL #这里用mem，内存型数据库
          driver-class-name: org.h2.Driver
```

通过以上两步，在springboot 测试类启动后，就会在H2数据库中产生测试用的数据。
