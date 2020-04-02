一个注解：
```
@Accessors(fluent = true)

//效果
Entity e = new Entity().name("a").address("b");
```
或
```
@Accessors(chain = true)

//效果
Entity e = new Entity().setAddress("beijing").setName("test");
```
