# 0x0 背景

为了尝试在mybatis框架下抽取BaseDao和BaseService，使用tk mapper作为通用mapper对dao层通用操作进行抽取。
按照作者的介绍：

>通用 Mapper4 是一个可以实现任意 MyBatis 通用方法的框架，项目提供了常规的增删改查操作以及Example 相关的单表操作。通用 Mapper 是为了解决 MyBatis 使用中 90% 的基本操作，使用它可以很方便的进行开发，可以节省开发人员大量的时间。

# 0x1 抽取BaseDao和BaseService
这里我们使用Springboot项目进行说明，maven依赖如下：
```
<!-- 数据库相关依赖 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>${postgresql.version}</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>tk.mybatis</groupId>
            <artifactId>mapper-spring-boot-starter</artifactId>
            <version>${tk.mybatis.version}</version>
        </dependency>
```

springboot启动类：
```
@SpringBootApplication
@MapperScan(basePackages= {"com.fly.ship.hiker.user.dao"})
@EnableSwagger2
public class HikerApplication {
    public static void main(String[] args) {
        SpringApplication.run(HikerApplication.class, args);
    }
}
```


###1.1 首先是dao层抽取：
```
public interface BaseDao<T> extends Mapper<T>, InsertListMapper<T> {}
```
其中Mapper包含了基本的增删改查，例如：
```
insertSelective(t);
delete(t);
updateByPrimaryKey(t);
selectByPrimaryKey(id);
...
```
InsertListMapper则包含了批量插入：
```
    @Options(useGeneratedKeys = true, keyProperty = "id")
    @InsertProvider(type = SpecialProvider.class, method = "dynamicSQL")
    int insertList(List<? extends T> recordList);
```
### 1.2 然后对service层抽取
代码如下：
```
//接口：
public interface BaseService<T, Id> {

    int save(T t);

    int delete(T t);

    int deleteById(Id id);

    int update(T t);

    T getById(Id id);

    List<T> getAll();

    int saveAll(List<T> list);

    List<T> searchByExample(Example example);

}

//实现类
public abstract class BaseServiceImpl<T, Id> implements BaseService<T, Id> {

    protected abstract BaseDao<T> getDao();

    @Override
    @Transactional
    public int save(T t) {
        return getDao().insertSelective(t);
    }

    @Override
    @Transactional
    public int saveAll(List<T> list) {
        return getDao().insertList(list);
    }

    @Override
    @Transactional
    public int delete(T t) {
        return getDao().delete(t);
    }

    @Override
    @Transactional
    public int deleteById(Id id) {
        return getDao().deleteByPrimaryKey(id);
    }

    @Override
    @Transactional
    public int update(T t) {
        return getDao().updateByPrimaryKey(t);
    }

    @Override
    public T getById(Id id) {
        return getDao().selectByPrimaryKey(id);
    }

    @Override
    public List<T> getAll() {
        return getDao().selectAll();
    }

    @Override
    public List<T> searchByExample(Example example) {
        return Collections.emptyList();
    }
}
```
# 0x3 测试
我们做一个User类对应于数据库中的user表
```
@Table(name = "public.user")
public class User {
    @Id
    private Integer id;

    private String username;

    private String password;

    private String name;

    @Column(name = "nick_name")
    private String nickName;
    //省略getters&setters
}
```
做一个UserDao
```
@Repository
public interface UserDao extends BaseDao<User> {
}
```
做一个UserService
```
//接口
public interface UserService extends BaseService<User, Integer> {
}

//实现类，注意这里的继承、实现写法
@Service
public class UserServiceImpl extends BaseServiceImpl<User, Integer> implements UserService {
    @Autowired
    private UserDao userDao;

    @Override
    protected BaseDao<User> getDao() {
        return userDao;
    }
}
```
最后来一个controller试下效果：
```
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @ApiOperation("测试")
    @GetMapping("/hello/{name}")
    public Result<String> hello(@PathVariable String name) {
        return new Result<>(200, "success", "test");
    }

    @ApiOperation("添加用户")
    @PostMapping
    public Result<Integer> saveUser(@RequestBody User user) {
        int count = userService.save(user);
        return new Result<>(200, "success", count);
    }

}
```
启动后swagger界面：

![Snipaste_2018-08-28_10-27-57.jpg](https://upload-images.jianshu.io/upload_images/13277366-044a1332db04caee.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

保存一个用书试试，发现控制台报错：
>java.lang.NoSuchMethodException: tk.mybatis.mapper.provider.base.BaseInsertProvider.<init>()
	at java.lang.Class.getConstructor0(Class.java:3082) ~[na:1.8.0_171]
	at java.lang.Class.newInstance(Class.java:412) ~[na:1.8.0_171]
	at org.apache.ibatis.builder.annotation.ProviderSqlSource.invokeProviderMethod(ProviderSqlSource.java:165) ~[mybatis-3.4.6.jar:3.4.6]
	at org.apache.ibatis.builder.annotation.ProviderSqlSource.createSqlSource(ProviderSqlSource.java:116) ~[mybatis-3.4.6.jar:3.4.6]
	at org.apache.ibatis.builder.annotation.ProviderSqlSource.getBoundSql(ProviderSqlSource.java:102) ~[mybatis-3.4.6.jar:3.4.6]
	at org.apache.ibatis.mapping.MappedStatement.getBoundSql(MappedStatement.java:292) ~[mybatis-3.4.6.jar:3.4.6]
	at org.apache.ibatis.executor.statement.BaseStatementHandler.<init>(BaseStatementHandler.java:64) ~[mybatis-3.4.6.jar:3.4.6]
	at org.apache.ibatis.executor.statement.PreparedStatementHandler.<init>(PreparedStatementHandler.java:40) ~[mybatis-3.4.6.jar:3.4.6]
......


百度一下，发现是由于启动类中MapperScan要用tk自带的，修改如下：
```
import org.mybatis.spring.annotation.MapperScan;
改为
import tk.mybatis.spring.annotation.MapperScan;
```
紧接着又报错：
>org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'baseDao' defined in file [C:\Users\guoxiang6\IdeaProjects\ship\target\classes\com\fly\ship\hiker\base\dao\BaseDao.class]: Invocation of init method failed; nested exception is tk.mybatis.mapper.MapperException: tk.mybatis.mapper.MapperException: java.lang.ClassCastException: sun.reflect.generics.reflectiveObjects.TypeVariableImpl cannot be cast to java.lang.Class
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1699) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:573) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]

看起来baseDao不能被MapperScan扫描到，于是将
```
@MapperScan(basePackages= {"com.fly.ship.hiker.**.dao"})
改为具体
@MapperScan(basePackages= {"com.fly.ship.hiker.user.dao"})
```
这下就好了。


