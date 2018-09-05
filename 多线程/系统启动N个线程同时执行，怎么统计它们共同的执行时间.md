## 多线程 - 系统启动N个线程同时执行，怎么统计它们共同的执行时间？

#### Java 技术栈星球上第一个作业

##### 系统启动N个线程同时执行，怎么统计它们共同的执行时间？你们知道几种方案？

```java
class TaskRunnable implements Runnable {
    @Override
    public void run() {
        long sTime = System.currentTimeMillis();
        long eTime = 0;
        try {
            Thread.sleep(1000);
            eTime = System.currentTimeMillis();
            log.info("Thread,id:{},name:{},time:{}", Thread.currentThread().getId(),
                    Thread.currentThread().getName(), eTime - sTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```



```java
/**
 * 方法一
 * awaitTermination 阻塞方法 美: [.tɜrmɪ'neɪʃ(ə)n]
 */
@Test
public void testCountTime1() throws InterruptedException {
    long startTime = System.currentTimeMillis();//主线程开始时间
    int threadCount = 5;
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);
    for (int i = 0; i < threadCount; i++) {
        executor.execute(new TaskRunnable());
    }
    executor.shutdown();

    /**
     * Blocks until all tasks have completed execution after a shutdown
     * request, or the timeout occurs, or the current thread is
     * interrupted, whichever happens first.
     */
    executor.awaitTermination(10, TimeUnit.SECONDS);
    long endTime = System.currentTimeMillis();//主线程结束时间
    log.info("主线程用时：{}ms", endTime - startTime);
}
16:47:06.811 [pool-1-thread-1] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:12,name:pool-1-thread-1,time:1000
16:47:06.811 [pool-1-thread-5] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:16,name:pool-1-thread-5,time:1000
16:47:06.811 [pool-1-thread-3] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:14,name:pool-1-thread-3,time:1000
16:47:06.811 [pool-1-thread-4] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:15,name:pool-1-thread-4,time:1000
16:47:06.811 [pool-1-thread-2] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:13,name:pool-1-thread-2,time:1000
16:47:06.815 [main] INFO com.example.demo.thread.chapter1.TaskRunnable - 主线程用时：1008ms
```



```java
/**
* 方法三： CountDownLatch   美: [lætʃ] 
*/
@Test
    public void testCountTime3() {
        long startTime = System.currentTimeMillis();//主线程开始时间
        int threadCount = 5;//线程数
        CountDownLatch cdl = new CountDownLatch(threadCount);
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        for (int i = 0; i < threadCount; i++) {
            executor.execute(() -> {
                long sTime = System.currentTimeMillis();
                long eTime = 0;
                try {
                    Thread.sleep(1000);
                    eTime = System.currentTimeMillis();
                    log.info("Thread,id:{},name:{},time:{}", Thread.currentThread().getId(),
                            Thread.currentThread().getName(), eTime - sTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    cdl.countDown();
                }
            });
        }
        try {
            cdl.await();
            long endTime = System.currentTimeMillis();//主线程结束时间
            log.info("主线程用时：{}ms", endTime - startTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
18:00:43.451 [pool-1-thread-5] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:16,name:pool-1-thread-5,time:1000
18:00:43.451 [pool-1-thread-3] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:14,name:pool-1-thread-3,time:1000
18:00:43.451 [pool-1-thread-2] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:13,name:pool-1-thread-2,time:1000
18:00:43.451 [pool-1-thread-4] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:15,name:pool-1-thread-4,time:1000
18:00:43.451 [pool-1-thread-1] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:12,name:pool-1-thread-1,time:1000
18:00:43.454 [main] INFO com.example.demo.thread.chapter1.TaskRunnable - 主线程用时：1049ms

/**
* 某一线程在开始运行前等待n个线程执行完毕。将CountDownLatch的计数器初始化为n new *CountDownLatch(n) ，每当一个任务线程执行完毕，就将计数器减1 countdownlatch.countDown()，当计数*器的值变为0时，在CountDownLatch上 await() 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主*线程需要等待多个组件加载完毕，之后再继续执行。
*/
```





```java
/**
     * 方法四
     * join
     * @author Andrew.Lin
     */
    @Test
    public void testCountTime4() throws InterruptedException {
        long startTime = System.currentTimeMillis();//主线程开始时间
        List<Thread> startList = new LinkedList();
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new TaskRunnable());
            t.start();
            startList.add(t);
        }

        startList.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        long endTime = System.currentTimeMillis();//主线程结束时间
        log.info("主线程用时：{}ms", endTime - startTime);
    }

18:07:00.730 [Thread-3] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:15,name:Thread-3,time:1000
18:07:00.730 [Thread-1] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:13,name:Thread-1,time:1000
18:07:00.730 [Thread-2] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:14,name:Thread-2,time:1000
18:07:00.730 [Thread-4] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:16,name:Thread-4,time:1000
18:07:00.730 [Thread-0] INFO com.example.demo.thread.chapter1.TaskRunnable - Thread,id:12,name:Thread-0,time:1000
18:07:00.736 [main] INFO com.example.demo.thread.chapter1.TaskRunnable - 主线程用时：1007ms

 /**join的意思是使得放弃当前线程的执行，并返回对应的线程，例如下面代码的意思就是：
         程序在main线程中调用t1线程的join方法，则main线程放弃cpu控制权，并返回t1线程继续执行直到线程t1执行完毕
         所以结果是t1线程执行完后，才到主线程执行，相当于在main线程中同步t1线程，t1执行完了，main线程才有执行的机会
         */
```