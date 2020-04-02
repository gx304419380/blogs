今天排查问题，发现读锁和写锁都被阻塞，而阻塞线程只持有读锁！

写demo模拟后出现了相同的问题：
```
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
        new Thread(() -> {
            lock.readLock().lock();
            sleep(120);
        }).start();


        new Thread(() -> {
            lock.writeLock().lock();
            sleep(120);
        }).start();

        lock.readLock().lock();
        System.out.println("finish");
```

打断点后，进入```readLock().lock()```方法，查看代码：
```
protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            //如果当前有写锁，并且写锁不是本线程，则返回-1
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            //关键在这里，判断读线程是否应该阻塞！
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //......
            }
            return fullTryAcquireShared(current);
        }
```
继续跟进```readerShouldBlock```方法，可以看出端倪
```
final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            //大概翻译一下可知：
            //这个方法是为了避免写锁无限等待而采取的一种启发式方法；
            //当等待队列中的头部（如果存在的话）正好是获取写锁的线程，则阻塞；
            return apparentlyFirstQueuedIsExclusive();
}
```
由此可知，当有写锁在等待队列头部时，其他读锁都要阻塞！如果某个读线程阻塞了，而此时有写线程去获取锁，会被放入等待队列头部；之后如果再次有其他读线程，则全都会阻塞！这个设计感觉不是很合理...
