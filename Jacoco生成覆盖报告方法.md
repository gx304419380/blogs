0x1 设置运行参数
==
```-javaagent:/jacocoagent.jar=output=tcpserver,port=8081,address=127.0.0.1,append=true```

其中/jacocoagent.jar是你的jacoco的路径；
后面的参数可以开启一个监听端口，通过这个端口和ip地址来获取当前正在运行项目的代码覆盖率exec文件（dump文件）。

0x2 获取dump文件
==
通过ant获取dump文件（也可以正常关闭项目，会自动在根目录生成dump文件）
```
<target name="dump">
  <jacoco:dump address="127.0.0.1" reset="false" destfile="${jacocoexecPath}" port="8081" append="true"/>
</target>
```
通过执行```ant dump```命令，可以生成dump文件到指定的路径${jacocoexecPath}；
dump文件是exec结尾的一个二进制文件，通过dump文件我们可以生成jacoco报告。

0x3 单元测试生成dump文件
==
以idea为例，如图所示，在运行参数中加入-javaagent配置即可：
![image.png](https://upload-images.jianshu.io/upload_images/13277366-a7ff2cfb31b44f29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当单元测试运行完后，会在对应的文件夹下生成相应的dump文件。

0x4 合并dump文件，并生成报告
==
ant中配置任务：
```
   <target name="merge">
        <jacoco:merge destfile="${execPath}">
            <fileset dir="${execPath}" includes="*.exec" />
        </jacoco:merge>
    </target>
```
执行 ``` ant merge ```命令，可以将指定文件夹${execPath}下的exec文件合并；

接下来配置生成报告任务，然后执行ant report来生成报告：
```
<target name="report">
  <!-- 报告路径 -->
  <delete dir="${reportfolderPath}" /> 
  <mkdir dir="${reportfolderPath}" />  

  <jacoco:report>
      <executiondata>
          <!-- dump文件路径 -->
          <file file="${execPath}" />
      </executiondata>

      <structure name="JaCoCo Report">
          <group name="scs-web">     
              <classfiles>
                  <!-- 覆盖率相关类路径，可以配置过滤规则 -->
                  <fileset dir="${webClasspath}" >
                    <include name="com/run/**/*.class"/>
                    <exclude name="com/test/**/*.class"/>
                  </fileset>
              </classfiles>
                <sourcefiles encoding="UTF-8">
                  <fileset dir="${webSrcpath}" />
              </sourcefiles>
          </group>
      </structure> 

      <html destdir="${reportfolderPath}" encoding="UTF-8" />         
  </jacoco:report>
</target>
```
