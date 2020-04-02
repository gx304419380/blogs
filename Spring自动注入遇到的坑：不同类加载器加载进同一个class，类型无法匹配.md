#0x0 背景
最近在测试从项目外部加载class进来，然后通过BeanFactoryPostProcessor注入到spring容器中，结果出了一点问题：**外部加载的class中@Autoware和@Resource注解都无法注入属性！**
报错如下：
>Caused by: org.springframework.beans.factory.BeanNotOfRequiredTypeException: Bean named 'testService' is expected to be of type 'com.fly.architecture.service.TestService' but was actually of type 'com.fly.architecture.service.TestService'
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:395) ~[spring-beans-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	
可以看到，所需注入的属性类是```com.fly.architecture.service.TestService```，实际注入的属性类是```com.fly.architecture.service.TestService```，看起来完全一样！

完整代码如下：
```
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    private static final String CLASS_PATH = "C:\\Users\\code\\";

    private Class<?> compileBeanClass() throws IOException, ClassNotFoundException {
        //自定义一个类加载器，加载class
        URL url = new URL("file:///" + CLASS_PATH );
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{url});
        return urlClassLoader.loadClass("TestController");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("调用了自定义的BeanFactoryPostProcessor加载bean");

        BeanDefinitionRegistry registry = (BeanDefinitionRegistry)beanFactory;

        Class<?> beanClass;
        try {
            beanClass = compileBeanClass();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
            throw new BeanCreationException(e.getMessage());
        }
        //将加载的class设置为bean definition，然后注入到spring容器中
        AnnotatedGenericBeanDefinition beanDefinition = new AnnotatedGenericBeanDefinition(beanClass);

        registry.registerBeanDefinition(beanClass.getSimpleName(), beanDefinition);
    }
}
```

#0x1 原因
进入debug，从报错的地方下断点，发现所需的类和bean的类虽然名字相同，但是Class对象却不是同一个：
![image.png](https://upload-images.jianshu.io/upload_images/13277366-3679d626129b6430.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从图中可以看出，两个对象虽然名称一致，但是hashCode以及内部的ClassLoader都不相同，因此spring在判断注入的bean类型和所需类型是否相同时，发现两者不一致！

#0x2 解决方法
将类加载的地方，改成如下方式，利用class loader的上级委托机制，实现不重复加载相同类：
```
        //正确做法，先获取当前类的classloader
        ClassLoader classLoader = getClass().getClassLoader();
        //这里很重要，需要设置当前classloader为父加载器，否则会加载重复的类进来
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{url}, classLoader);
        
        return urlClassLoader.loadClass("TestController");
```
