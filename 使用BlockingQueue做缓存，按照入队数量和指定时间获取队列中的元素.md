直接上代码：
```
private static final Integer CLEAR_CACHE_INTERVAL = 1000;

private static final BlockingQueue<PictureUrlDto> CACHE_QUEUE = new ArrayBlockingQueue<>(CACHE_SIZE);
```
下面是调用Queues.drain方法从队列中拿数据
```
List<PictureUrlDto> data = new ArrayList<>(CACHE_SIZE);
        try {
            while (true) {
                //清空data
                data.clear();
                //将缓存中的数据放入data，规则：缓存数据超过缓存大小的0.75或者时间超过1000ms
                Queues.drain(CACHE_QUEUE, data, CACHE_SIZE * 3 >> 2, CLEAR_CACHE_INTERVAL, TimeUnit.MILLISECONDS);

                //执行批量入库
                if (ObjectUtils.isNotEmpty(data)) {
                    pictureTaskService.savePictureUrlList(data);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            log.error(PICTURE_TASK_POLL_CACHE_ERROR);
        }
```
