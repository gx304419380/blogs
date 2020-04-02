现在没时间，先把代码贴上：
```
package com.fly.architecture.code;

import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;

import javax.tools.*;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.io.OutputStream;
import java.lang.reflect.Method;
import java.net.URI;
import java.net.URL;
import java.net.URLClassLoader;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.*;
import java.util.stream.Collectors;

/**
 * <p></p>
 *
 */
@Slf4j
public class CompileUtils {
    public static final String JAVA_CODE = 
            "public class Man {\n" +
            "\tpublic void hello(){\n" +
            "\t\tSystem.out.println(\"hello world\");\n" +
            "\t}\n" +
            "}";

    public static void main(String[] args) throws Exception {
        String sourcePath = "C:\\Users\\IdeaProjects\\architecture\\src\\main\\resources\\code";
        ClassLoader classLoader = compile(sourcePath, true);

        Class<?> aClass = classLoader.loadClass("com.fly.HelloWorld");
        Object instance = aClass.newInstance();
        Method hello = aClass.getMethod("hello");
        hello.invoke(instance);
    }


    /**
     * @param sourcePath        源码路径
     * @param generateClassFile 是否生成类文件
     * @return
     * @throws IOException
     */
    public static ClassLoader compile(String sourcePath, boolean generateClassFile) throws IOException {


        Path root = Paths.get(sourcePath);
        List<String> files = Files.walk(root)
                .filter(path -> path.toFile().isFile())
                .filter(path -> path.toString().endsWith(".java"))
                .map(Path::toString)
                .collect(Collectors.toList());

        JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
        StandardJavaFileManager standardFileManager = compiler.getStandardFileManager(null, null, null);
        ClassJavaFileManager memoryJavaFileManager = new ClassJavaFileManager(standardFileManager);
        Iterable<? extends JavaFileObject> javaFileObjects = standardFileManager.getJavaFileObjectsFromStrings(files);

        JavaFileObject javaFileObject = new MemoryJavaFileObject("Man", JAVA_CODE);
        ArrayList<JavaFileObject> list = Lists.newArrayList(javaFileObjects);
        list.add(javaFileObject);

        ArrayList<String> options = new ArrayList<>();
        JavaFileManager manager = memoryJavaFileManager;

        //是否生成class文件
        String targetPath = sourcePath + File.separator + "target_dir";
        if (generateClassFile) {
            manager = standardFileManager;
            generateClassFile(options, targetPath);
        }

        JavaCompiler.CompilationTask task = compiler.getTask(null, manager, null, options, null, list);
        Boolean taskSuccess = task.call();
        System.out.println("compile " + (taskSuccess ? "success" : "fail"));

        standardFileManager.close();
        memoryJavaFileManager.close();

        if (generateClassFile) {
            URL url = Paths.get(targetPath).toFile().toURI().toURL();
            return new URLClassLoader(new URL[]{url}, CompileUtils.class.getClassLoader());
        }
        return new MemoryClassLoader(memoryJavaFileManager, CompileUtils.class.getClassLoader());
    }

    /**
     * 生成类文件到磁盘
     * @param options       编译命令行
     * @param targetPath    目标路径
     * @throws IOException
     */
    public static void generateClassFile(ArrayList<String> options, String targetPath) throws IOException {
        options.add("-d");
        options.add(targetPath);
        //清空输出路径
        Path target = Paths.get(targetPath);
        if (Files.exists(target)) {
            Files.walk(target)
                    .sorted(Comparator.reverseOrder())
                    .map(Path::toFile)
                    .forEach(File::delete);
        }

        Files.createDirectory(target);
    }


    /**
     * 自定义fileManager
     */
    static class ClassJavaFileManager extends ForwardingJavaFileManager<JavaFileManager> {


        final Map<String, JavaFileObject> classMap = new HashMap<>();

        public ClassJavaFileManager(JavaFileManager fileManager) {
            super(fileManager);
        }

        public Map<String, JavaFileObject> getClassMap() {
            return classMap;
        }

        //这个方法一定要自定义
        @Override
        public JavaFileObject getJavaFileForOutput(Location location, String className, JavaFileObject.Kind kind, FileObject sibling) throws IOException {
            JavaFileObject classJavaFileObject = new MemoryJavaFileObject(className, kind);
            classMap.put(className, classJavaFileObject);
            return classJavaFileObject;
        }
        
    }


    /**
     * 自定义的JavaFileObject，包含源码和字节码byte[]
     */
    public static class MemoryJavaFileObject extends SimpleJavaFileObject {
        private String source;
        private ByteArrayOutputStream outPutStream;
        // 该构造器用来输入源代码
        public MemoryJavaFileObject(String name, String source) {
            // 1、先初始化父类，由于该URI是通过类名来完成的，必须以.java结尾。
            // 2、如果是一个真实的路径，比如是file:///test/demo/Hello.java则不需要特别加.java
            // 3、这里加的String:///并不是一个真正的URL的schema, 只是为了区分来源
            super(URI.create("String:///" + name + Kind.SOURCE.extension), Kind.SOURCE);
            this.source = source;
        }
        // 该构造器用来输出字节码
        public MemoryJavaFileObject(String name, Kind kind){
            super(URI.create("String:///" + name.replace(".", "/") + kind.extension), kind);
            source = null;
        }

        @Override
        public CharSequence getCharContent(boolean ignoreEncodingErrors){
            if(source == null){
                throw new IllegalArgumentException("source == null");
            }
            return source;
        }

        @Override
        public OutputStream openOutputStream() throws IOException {
            outPutStream = new ByteArrayOutputStream();
            return outPutStream;
        }

        // 获取编译成功的字节码byte[]
        public byte[] getCompiledBytes(){
            return outPutStream.toByteArray();
        }
    }


    //自定义classloader
    static class MemoryClassLoader extends URLClassLoader {

        // class name to class bytes:
        Map<String, JavaFileObject> classBytes = new HashMap<>();

        public MemoryClassLoader(ClassJavaFileManager manager, ClassLoader classLoader) {
            super(new URL[0], classLoader);
            this.classBytes.putAll(manager.getClassMap());
        }

        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            MemoryJavaFileObject classObject = (MemoryJavaFileObject) classBytes.get(name);
            if (classObject == null) {
                return super.findClass(name);
            }
            classBytes.remove(name);
            byte[] bytes = classObject.getCompiledBytes();
            return defineClass(name, bytes, 0, bytes.length);
        }

    }

}
```
