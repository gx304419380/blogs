0x1 背景
==
在开发过程中，经常遇到发送事件来通知其他模块进行相应的业务处理；笔者实用的是spring自带的```ApplicationEventPublisher```和```EventListener```进行事件的发收；
但是开发时遇到一个问题：
如果事件很多，但是事件模式都差不多，就需要定义很多事件类来分别表示各种事件，例如，我们进行数据同步，每同步一条数据都要发送对应的事件，伪代码如下：
```
//事件类
class RegionEvent {
  private Region region;
  private OperationEnum operation;
}

class UserEvent {
  private User user;
  private OperationEnum operation;
}

//插入一个区域
regionDao.insert(Region region);
//发送插入区域事件
publisher.publishEvent(new RegionEvent(region, INSERT));

//更新一个用户
userDao.update(User user);
//发送更新用户事件
publisher.publishEvent(new UserEvent(user, UPDATE));

//区域事件监听器
@EventListener
public void onRegionEvent(RegionEvent event) {
    log.info("receive event: {}", event);
}

//用户事件监听器
@EventListener
public void onUserEvent(UserEvent event) {
    log.info("receive event: {}", event);
}
```
此时，我们发现有太多冗余的代码，因为每插入一种类型的数据，就要对应的建立一个和该类型相关的事件类；自然而然地，我们想到可以使用泛型来简化以上逻辑。

0x1 泛型事件遇到的问题
==
我们定义一种泛型事件，来重新实现以上的逻辑，此时我们发现一个问题：**发送的事件根本监听不到**，伪代码如下：
```
class BaseEvent<T> {
  private T data;
  private OperationEnum operation;
}

//发送插入区域事件
publisher.publishEvent(new BaseEvent<>(region, INSERT));
//发送更新用户事件
publisher.publishEvent(new BaseEvent<>(user, UPDATE));

//区域事件监听器
@EventListener
public void onRegionEvent(BaseEvent<Region> event) {
    log.info("receive event: {}", event);
}

//用户事件监听器
@EventListener
public void onUserEvent(BaseEvent<User> event) {
    log.info("receive event: {}", event);
}
```
**这是由于spring在解析事件类型时，并没有对事件的泛型进行解析，导致在运行时所有publish的事件都被spring解析成了BaseEvent<?>事件**，如果采用如下代码，则会监听到所有事件：
```
@EventListener
public void onUserEvent(BaseEvent<Object> event) {
    log.info("receive event: {}", event);
}

@EventListener
public void onUserEvent(BaseEvent event) {
    log.info("receive event: {}", event);
}
```
0x2 解决方法
==
查阅了spring的文档后，发现spring已经考虑到这一点，官方文档原文如下：
>In certain circumstances, this may become quite tedious if all events follow the same structure. In such a case, you can implement ResolvableTypeProvider to guide the framework beyond what the runtime environment provides. The following event shows how to do so:
大概翻译一下：
在某些情况下，如果所有事件类型都遵循相同的结构，这会是特别恶心的一件事。在这种情况下，你可以通过实现ResolvableTypeProvider接口，在运行时基于环境提供的信息来引导框架

我们基于spring提供的方法，对原有的泛型事件进行改造：

```
public class BaseEvent<T> implements ResolvableTypeProvider {
  private T data;
  private OperationEnum operation;

  @Override
  public ResolvableType getResolvableType() {
      return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forClass(getData().getClass()));
  }
}
```
此时，使用上文的监听器就可以监听到对应的事件了；

0x3 原理
==
事件监听器和事件是通过事件类型进行匹配的，而事件类型的publish源码在AbstractApplicationContext类的
```protected void publishEvent(Object event, @Nullable ResolvableType eventType)```
方法中，如下：
```
        ApplicationEvent applicationEvent;
        
		if (event instanceof ApplicationEvent) {
            //对于继承ApplicationEvent的事件，
			applicationEvent = (ApplicationEvent) event;
		}
		else {
            //对于非继承ApplicationEvent的事件，包装成PayloadApplicationEvent，
            //然后通过getResolvableType()获取事件类型
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

```
然后进入``` multicastEvent(applicationEvent, eventType)```方法：

```
	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        //这里对于ApplicationEvent的子类事件，进行解析事件类型
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
        //根据上面解析到的eventType，获取对应的监听器，并依次执行回调方法
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

可以发现，关键在于如何解析事件类型，分别进入上文中```resolveDefaultEventType()```方法和```getResolvableType()```方法，可以看到解析事件类型的具体细节如下：
```
//针对PayloadApplicationEvent，通过下面的方法处理，可见
	@Override
	public ResolvableType getResolvableType() {
		return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getPayload()));
	}
//对于继承了ApplicationEvent的事件类
	private ResolvableType resolveDefaultEventType(ApplicationEvent event) {
		return ResolvableType.forInstance(event);
	}
```

上述两个方法用于根据事件构造事件的ResolvableType，关键代码在```ResolvableType.forInstance()```:
```
	public static ResolvableType forInstance(Object instance) {
		Assert.notNull(instance, "Instance must not be null");
		if (instance instanceof ResolvableTypeProvider) {
			ResolvableType type = ((ResolvableTypeProvider) instance).getResolvableType();
			if (type != null) {
				return type;
			}
		}
		return ResolvableType.forClass(instance.getClass());
	}
```
至此，可以看到，如果事件实现了ResolvableTypeProvider接口，则可以通过调用getResolvableType方法获取事件的带泛型类型，如果未实现该接口，则只能获取事件的原始类型，效果如下：

未实现接口的情况下：
![image.png](https://upload-images.jianshu.io/upload_images/13277366-88a8d64ac832f0a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实现接口后：
![image.png](https://upload-images.jianshu.io/upload_images/13277366-9031e84a16fb2e41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
