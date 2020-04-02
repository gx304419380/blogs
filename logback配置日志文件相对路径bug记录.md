最近发现日志分割后不知道存在哪里了，检查了配置文件，发现：
```
<fileNamePattern>logs/ibp-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
```
这里用了相对路径，换成绝对路径后就好了。
或者，更一般的，使用占位符：
```
<fileNamePattern> ${catalina.base}/logs/ibp-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
```
