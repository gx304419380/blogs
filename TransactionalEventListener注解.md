0x0 背景
==
在项目中，往往需要执行数据库操作后，发送消息或事件来异步调用其他组件执行相应的操作，例如：
用户注册后发送激活码；
配置修改后发送更新事件等。
但是，数据库的操作如果还未完成，此时异步调用的方法查询数据库发现没有数据，这就会出现问题。伪代码如下：
```
void saveUser(User u) {
    //保存用户信息
    userDao.save(u);
    //触发保存用户事件
    applicationContext.publishEvent(new SaveUserEvent(u.getId()));
}

@EventListener
void onSaveUserEvent(SaveUserEvent event) {
    //获取事件中的信息（用户id）
    Integer id = event.getEventData();
    //查询数据库，获取用户（此时如果用户还未插入数据库，则返回空）
    User u = userDao.getUserById(id);
    //这里可能报空指针异常！
    String phone = u.getPhoneNumber();
    MessageUtils.sendMessage(phone);
}
```

0x1 解决方法
==
为了解决上述问题，Spring为我们提供了两种方式：
(1)  ```@TransactionalEventListener```注解
(2)  事务同步管理器```TransactionSynchronizationManager```

1.1 @TransactionalEventListener使用方法
--
仍旧是上述给用户发短信的例子，代码如下：
```
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
void onSaveUserEvent(SaveUserEvent event) {
    Integer id = event.getEventData();
    User u = userDao.getUserById(id);
    String phone = u.getPhoneNumber();
    MessageUtils.sendMessage(phone);
}
```
这样，只有当前事务提交之后，才会执行事件监听器的方法。其中参数phase默认为AFTER_COMMIT，共有四个枚举：
```
/**
	 * Fire the event before transaction commit.
	 * @see TransactionSynchronization#beforeCommit(boolean)
	 */
	BEFORE_COMMIT,

	/**
	 * Fire the event after the commit has completed successfully.
	 * <p>Note: This is a specialization of {@link #AFTER_COMPLETION} and
	 * therefore executes in the same after-completion sequence of events,
	 * (and not in {@link TransactionSynchronization#afterCommit()}).
	 * @see TransactionSynchronization#afterCompletion(int)
	 * @see TransactionSynchronization#STATUS_COMMITTED
	 */
	AFTER_COMMIT,

	/**
	 * Fire the event if the transaction has rolled back.
	 * <p>Note: This is a specialization of {@link #AFTER_COMPLETION} and
	 * therefore executes in the same after-completion sequence of events.
	 * @see TransactionSynchronization#afterCompletion(int)
	 * @see TransactionSynchronization#STATUS_ROLLED_BACK
	 */
	AFTER_ROLLBACK,

	/**
	 * Fire the event after the transaction has completed.
	 * <p>For more fine-grained events, use {@link #AFTER_COMMIT} or
	 * {@link #AFTER_ROLLBACK} to intercept transaction commit
	 * or rollback, respectively.
	 * @see TransactionSynchronization#afterCompletion(int)
	 */
	AFTER_COMPLETION
```

1.2  TransactionSynchronizationManager方法
--
仍旧是上述案例，代码如下：
```
@EventListener
void onSaveUserEvent(SaveUserEvent event) {
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            Integer id = event.getEventData();
            User u = userDao.getUserById(id);
            String phone = u.getPhoneNumber();
            MessageUtils.sendMessage(phone);
        }
    });
}
```
其实，@TransactionalEventListener底层也是这样实现的。
