没时间写细节，直接上代码

```


/**
 * <p>集群模式下的切面</p>
 * @since 
 */
@Component
@Aspect
@Order(1)
@ConditionalOnExpression("${resource.sync.cluster:false}==true")    //只有当集群模式下该切面才生效
@Slf4j
public class ClusterAspect {

    private RLock redisLock;
    //由于改变量被多个线程读写，所以上读写锁，保障任务执行完成后，才能改变变量值
    private volatile boolean leader;

    private static final ReentrantReadWriteLock LEADER_READ_WRITE_LOCK = new ReentrantReadWriteLock();
    private static final ReentrantReadWriteLock.ReadLock LEADER_READ_LOCK = LEADER_READ_WRITE_LOCK.readLock();
    private static final ReentrantReadWriteLock.WriteLock LEADER_WRITE_LOCK = LEADER_READ_WRITE_LOCK.writeLock();

    private static volatile boolean LOOP_FLAG = true;

    private static final String LOCK_KEY = "CLUSTER_LEADER";

    //选举间隔，默认10000ms
    @Value("${resource.sync.cluster.elect.interval:10000}")
    private Integer electInterval;

    /**
     * 门栓，当首次获取redis锁执行后才能打开
     */
    public static final CountDownLatch LATCH = new CountDownLatch(1);

    @Autowired
    private TaskExecutor taskExecutor;

    /**
     * 利用redis模拟选主
     * 弊端: 主节点挂掉后，可能无法立即选出新主节点，20s后才有
     */
    @EventListener(ApplicationReadyEvent.class)
    @Async
    public void electLeader() {
        redisLock = SyncDistributedLockUtil.getLock(LOCK_KEY);
        //每隔10s拿一次锁，相当于10s选举一次，
        while (LOOP_FLAG) {
            //LEADER写锁
            LEADER_WRITE_LOCK.lock();
            try {
                //如果当前是主节点，放锁
                if (leader && redisLock.isHeldByCurrentThread()) {
                    redisLock.unlock();
                }
                //设置锁过期时间, 20s
                //重新获取锁，相当于选举过程
                leader = redisLock.tryLock(0L, 20L, TimeUnit.SECONDS);
            } catch (Exception e) {
                log.error("- get redis lock error!", e);
            } finally {
                LEADER_WRITE_LOCK.unlock();
            }

            //开门，放行启动同步
            if (LATCH.getCount() == 1L) {
                LATCH.countDown();
            }

            //每隔十秒放一次锁
            LockSupport.parkNanos(electInterval * 1000000L);
        }

        //项目关闭记得放锁
        if (leader && redisLock.isHeldByCurrentThread()) {
            log.debug("- project is shutting down, release the redis lock...");
            redisLock.unlock();
        }
    }

    @PreDestroy
    public void destroy() {
        endLoop();
    }


    @Around("@annotation(syncPreProcessor)")
    public Object clusterModel(ProceedingJoinPoint joinPoint, SyncPreProcessor syncPreProcessor) {

        RedisExtendTask redisExtendTask = new RedisExtendTask(redisLock);

        try {
            //等待首次获取选主执行完成
            LATCH.await();
            //判断是否为主节点
            log.info("- +++++++++++ entering cluster mode！ Check whether this node is Leader: {}！...", leader);
            //LEADER变量读锁
            LEADER_READ_LOCK.lock();
            //如果不是主节点，直接退出
            if (!leader) {
                log.info("- +++++++++++ not LEADER, return...");
                return null;
            }
            //是主节点，执行任务
            log.info("- +++++++++++ this node is LEADER, begin to sync resource ...");
            //一旦任务开始执行，redis锁不能放，启动redis锁延期线程
            taskExecutor.execute(redisExtendTask);
            return joinPoint.proceed();
        } catch (Throwable throwable) {
            log.error("- +++++++++++ error when doing sync job, message: {}", throwable.getMessage());
            throw new BaseRuntimeException(throwable.getMessage());
        } finally {
            //放锁
            LEADER_READ_LOCK.unlock();
            //停止redis锁延长线程
            redisExtendTask.endTask();
            log.debug("- +++++++++++ release LEADER_READ_LOCK: {} success ...");
        }
    }


    private static void endLoop() {
        LOOP_FLAG = false;
    }


    /**
     * redis lock 延时线程
     */
    private static class RedisExtendTask implements Runnable {

        private volatile boolean flag = true;

        private RLock redisLock;

        RedisExtendTask(RLock redisLock) {
            this.redisLock = redisLock;
        }

        @Override
        public void run() {
            log.debug("- begin to extend the expire time for redis lock: {}", redisLock.getName());
            //每秒延长一下过期时间，直到任务结束
            while (flag) {
                redisLock.expire(20L, TimeUnit.SECONDS);
                if (log.isDebugEnabled()) {
                    log.debug("- redis lock remain time to live = {}ms", redisLock.remainTimeToLive());
                }
                LockSupport.parkNanos(1000000000L);
            }
            log.debug("- finish extending the expire time for redis lock: {}", redisLock.getName());
        }

        void endTask() {
            this.flag = false;
        }
    }
}
```
