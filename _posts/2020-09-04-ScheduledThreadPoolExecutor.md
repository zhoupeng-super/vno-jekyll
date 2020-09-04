# ScheduledThreadPoolExecutor
### 1、介绍：
> 自JDK1.5开始，JDK提供了ScheduledThreadPoolExecutor类来支持周期性的调度，在这之前的实现需要Timer和TimerTask或者第三方工具完成。但Timer有很多缺陷。

>> - Timer是单线程的。
>> - 如果在执行任务期间，某个Timer耗时很久，则会影响其他Timer的调度。
>> - Timer的调度是基于绝对时间的，对系统时间敏感
>> - Timer不会捕获执行TimerTask抛出的异常，由于Timer是单线程的，所以一但出现问题，线程就会终止，其他任务也得不到执行。

> ScheduledThreadPoolExecutor继承ThreadPoolExecutor线程池，重用线程池的功能，它的实现方式如下：

>> - 将任务封装成ScheduledFutureTask对象，ScheduledFutureTask基于相对时间，不受系统时间干扰。
>> - ScheduledFutureTask继承FutureTask，可以通过Future对象获取执行结果。
>> - ScheduledFutureTask实现了java.util.concurrent.Delayed和java.lang.Comparable接口，所以有两个重要的方法：compareTo和getDelay方法。compareTo方法比较任务的优先级关系，如果距离下次执行的时间较短，则优先级高；getDelay方法用于返回距离下次任务执行的时间间隔。
>> - ScheduledThreadPoolExecutor定义了一个DelayedWorkQueue,它是一个有序队列，会通过每个任务下次执行时间的时间间隔大小来排序。

> 通过以上介绍，可以对比下Timer和ScheduledThreadPoolExecutor:

|                     Timer                      |      ScheduledThreadPoolExecutor       |
| :--------------------------------------------: | :------------------------------------: |
|                     单线程                     |                 多线程                 |
|       单个任务执行时间，影响其他任务调度       |             多线程不会影响             |
|                  基于绝对时间                  |              基于相对时间              |
| 一旦任务执行异常，不会捕获，其他任务得不到执行 | 多线程，单个任务的执行不会形象其他线程 |

### 2、ScheduledThreadPoolExecutor的使用

> 举个例子说明ScheduledThreadPoolExecutor的使用

``` java
public class ScheduledThreadPoolTest {
    public static void main(String[] args) throws InterruptedException {
        // 创建大小为5的线程池
        ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
        for (int i = 0; i < 3; i++) {
            Task worker = new Task("task-" + i);
            // 只执行一次
//          scheduledThreadPool.schedule(worker, 5, TimeUnit.SECONDS);
            // 周期性执行，每5秒执行一次
            scheduledThreadPool.scheduleAtFixedRate(worker, 0,5, TimeUnit.SECONDS);
        }
        Thread.sleep(10000);
        System.out.println("Shutting down executor...");
        // 关闭线程池
        scheduledThreadPool.shutdown();
        boolean isDone;
        // 等待线程池终止
        do {
            isDone = scheduledThreadPool.awaitTermination(1, TimeUnit.DAYS);
            System.out.println("awaitTermination...");
        } while(!isDone);
        System.out.println("Finished all threads");
    }
}
class Task implements Runnable {
    private String name;
    public Task(String name) {
        this.name = name;
    }
    @Override
    public void run() {
        System.out.println("name = " + name + ", startTime = " + new Date());
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("name = " + name + ", endTime = " + new Date());
    }
}
```

> 下面来分析下ScheduledThreadPoolExecutor的实现过程

### 3、ScheduledThreadPoolExecutor的实现过程

#### 3.1、ScheduledThreadPoolExecutor的类结构

![diagram](images/diagram.png)

>  其中包含了有关调度和队列的接口
>
> ScheduldedThreadPoolExecutor继承ThreadPoolExecutor，实现了ScheduledExecutorService接口，该接口定义了scheduled等任务调度的方法。
>
> 同时ScheduledThreadPoolExecutor有两个重要的内部类：DelayedWorkQueue和ScheduledFutureTask。可以看到DelayedWorkerQueue是一个阻塞队列，而ScheduledFutureTask继承FutureTask，并且实现了Delayed接口。

#### 3.2、ScheduledThreadPoolExecutor的类结构

> ScheduledThreadPoolExecutor中有四个构造方法；
>
> ````java
>     public ScheduledThreadPoolExecutor(int corePoolSize) {
>         super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
>               new DelayedWorkQueue());
>     }
> 
>     public ScheduledThreadPoolExecutor(int corePoolSize,
>                                        ThreadFactory threadFactory) {
>         super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
>               new DelayedWorkQueue(), threadFactory);
>     }
> 
>     public ScheduledThreadPoolExecutor(int corePoolSize,
>                                        RejectedExecutionHandler handler) {
>         super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
>               new DelayedWorkQueue(), handler);
>     }
> 
>     public ScheduledThreadPoolExecutor(int corePoolSize,
>                                        ThreadFactory threadFactory,
>                                        RejectedExecutionHandler handler) {
>         super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
>               new DelayedWorkQueue(), threadFactory, handler);
>     }
> ````
>
> 因为ScheduledThreadPoolExecutor继承ThreadPoolExecutor，所以这里都是调用的ThreadPoolExecutor的构造方法，注意这里传入的是DelayedWorkQueue类型的对象。

#### 3.3、schedule方法

> 在上文的例子中，使用schedule方法进行任务调度，源码如下：
>
> ````java
> public ScheduledFuture<?> schedule(Runnable command,
>                                        long delay,
>                                        TimeUnit unit) {
>         if (command == null || unit == null)
>             throw new NullPointerException();
>         RunnableScheduledFuture<?> t = decorateTask(command,
>             new ScheduledFutureTask<Void>(command, null,
>                                           triggerTime(delay, unit)));
>         delayedExecute(t);
>         return t;
>     }
> 
> public <V> ScheduledFuture<V> schedule(Callable<V> callable,
>                                            long delay,
>                                            TimeUnit unit) {
>         if (callable == null || unit == null)
>             throw new NullPointerException();
>         RunnableScheduledFuture<V> t = decorateTask(callable,
>             new ScheduledFutureTask<V>(callable,
>                                        triggerTime(delay, unit)));
>         delayedExecute(t);
>         return t;
>     }
> ````
>
> 这里两个重载的schedule方法只是第一个参数不同，可以是Runnable对象或者是Callable对象。会把传入的对象封装成RunnableScheduledFuture对象，其实也就是ScheduledFutureTask对象，decorateTask默认什么功能都没有做，子类可以重写该方法。
>
> ````java
> protected <V> RunnableScheduledFuture<V> decorateTask(
>         Callable<V> callable, RunnableScheduledFuture<V> task) {
>         return task;
>     }
> ````
>
> 然后，通过调用delayedExecute方法来延迟执行任务。
>
> 最后，返回一个ScheduledFuture对象。

#### 3.4、scheduleAtFixedRate方法

> 该方法设置了执行周期，下一次执行时间相当于上一次执行时间加上period，它是采用已固定的频率执行。
>
> ````java
> public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
>                                                   long initialDelay,
>                                                   long period,
>                                                   TimeUnit unit) {
>         if (command == null || unit == null)
>             throw new NullPointerException();
>         if (period <= 0)
>             throw new IllegalArgumentException();
>         ScheduledFutureTask<Void> sft =
>             new ScheduledFutureTask<Void>(command,
>                                           null,
>                                           triggerTime(initialDelay, unit),
>                                           unit.toNanos(period));
>         RunnableScheduledFuture<Void> t = decorateTask(command, sft);
>         sft.outerTask = t;
>         delayedExecute(t);
>         return t;
>     }
> ````

#### 3.5、scheduleWithFixedDelay方法

> 该方法设置了执行周期，与scheduledAtFixedRate方法不同的是，下一次执行时间是上一次任务执行完的系统时间加上period，因而具体执行时间不是固定的，但是周期是固定的，是采用相对固定的延迟周期来执行任务。
>
> ````java
> public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
>                                                      long initialDelay,
>                                                      long delay,
>                                                      TimeUnit unit) {
>         if (command == null || unit == null)
>             throw new NullPointerException();
>         if (delay <= 0)
>             throw new IllegalArgumentException();
>         ScheduledFutureTask<Void> sft =
>             new ScheduledFutureTask<Void>(command,
>                                           null,
>                                           triggerTime(initialDelay, unit),
>                                           unit.toNanos(-delay));
>         RunnableScheduledFuture<Void> t = decorateTask(command, sft);
>         sft.outerTask = t;
>         delayedExecute(t);
>         return t;
>     }
> ````
>
> 注意这里的unit.toNanos(-delay)，这里把周期设置为负数，来表示是相对固定的延迟执行。
>
> scheduleAtFixedRate和scheduleWithFixedDelay方法的区别就在这
>
> ````java
> private void setNextRunTime() {
>             long p = period;
>             if (p > 0) //固定频率，上次执行时间加上周期时间
>                 time += p;
>             else	//相对固定延迟执行，使用当前系统时间加上周期时间
>                 time = triggerTime(-p);
>         }
> ````
>
> setNextRunTime会在run方法中，执行完任务进行调用。

#### 3.6、triggerTime方法

> triggerTime方法用于获取下一次执行的具体时间。
>
> ````java
> private long triggerTime(long delay, TimeUnit unit) {
>         return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
>     }
> 
>     long triggerTime(long delay) {
>         return now() +
>             ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
>     }
> ````
>
> 这里的delay < (Long.MAX_VALUE >> 1) 是为了判断是否要防止long类型溢出，如果delay值<long类型值的一半，则会直接返回delay，否则需要进行防止溢出处理。

#### 3.7、overflowFree方法

> 该方法的作用是限制队列中所有节点的延迟时间在Long.MAX_VALUE之内，防止在compareTo方法中溢出。
>
> ````java
> private long overflowFree(long delay) {
>   			//获取队列中的第一个节点
>         Delayed head = (Delayed) super.getQueue().peek();
>         if (head != null) {
>           	//获取延迟时间
>             long headDelay = head.getDelay(NANOSECONDS);
>           	// 如果延迟时间小于0，并且 delay - headDelay 超过了Long.MAX_VALUE
>         		// 将delay设置为 Long.MAX_VALUE + headDelay 保证delay小于Long.MAX_VALUE
>             if (headDelay < 0 && (delay - headDelay < 0))
>                 delay = Long.MAX_VALUE + headDelay;
>         }
>         return delay;
>     }
> ````
>
> 当一个任务可以执行出队列操作，但是还是没有执行，可能由于线程池中的工作线程不是空闲的，具体分析下这个情况：
>
> - 为了方便说明，假设Long.MAX_VALUE=1023，也就是11位，并且当前的时间是100，调用tirggerTime并没有对delay进行判断，而是直接返回来now()+delay，也就相当于是100+1023，这肯定是溢出了，那么返回的时间是-925。
> - 如果头节点可以出对，但是还没有执行出对，那么头节点的执行时间应该是小于当前时间的，假设是95.
> - 这时调用offer方法向队列中添加任务，在offer方法中会调用siftUp方法来排序，在siftUp方法执行时，又会调用ScheduledFutureTask的compareTo方法来比较执行时间。
> - 这时如果执行到了compareTo方法中的long diff = time - x.time;时，那么计算后的结果就是-925-95=1020，那么将放回-1，而正常情况下应该返回1，因为新加入的任务要比头节点任务的执行时间要晚，这就不是我们想要的结果了，会导致队列中的顺序不正确。
> - 同理也可以算一下在执行compareTo方法中的long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);时也会出现这个问题。
> - 所以在trrigerTime中对delay的大小做了判断，就是为了防止这中情况发生。

> 如果执行了overflowFree方法呢，这时headDelay = 95 - 100 = -5，然后执行2013 + （-5）=1018，那么trriggerTime会返回100 + 1018 = -930。再执行compareTo方法中的`long diff = time - x.time;`时，`diff = -930 - 95 = -930 - 100 + 5 = 1018 + 5 = 1023`，没有溢出，符合正常的预期。
>
> 所以，overflowFree方法中把已经超时的部分时间给减去，就是为了避免在compareTo方法中出现溢出情况。
>
> （说实话，这段代码看的很痛苦，一般情况下也不会发生这种情况，谁会传一个Long.MAX_VALUE呢。要知道Long.MAX_VALUE的纳秒数换算成年的话是292年，谁会这么无聊。。。） 

#### 3.8、ScheduledFutureTash的getDelay方法

> ```java
> public long getDelay(TimeUnit unit) {
>   	//执行时间减去当前系统时间
>     return unit.convert(time - now(), NANOSECONDS);
> }
> ```

#### 3.9、ScheduledFutureTask的构造方法

> ScheduledFutureTask继承自FutureTask，实现了RunnableFutureTask接口，具体参照上文。
>
> ```java
> ScheduledFutureTask(Runnable r, V result, long ns) {
>     super(r, result);
>     this.time = ns;
>     this.period = 0;
>     this.sequenceNumber = sequencer.getAndIncrement();
> }
> 
> ScheduledFutureTask(Runnable r, V result, long ns, long period) {
>     super(r, result);
>     this.time = ns;
>     this.period = period;
>     this.sequenceNumber = sequencer.getAndIncrement();
> }
> 
> ScheduledFutureTask(Callable<V> callable, long ns) {
>     super(callable);
>     this.time = ns;
>     this.period = 0;
>     this.sequenceNumber = sequencer.getAndIncrement();
> }
> ```
>
> 这里有几个重要的属性，下面解释下：
>
> ```java
> //下次任务执行的时间
> this.time = ns;
> //任务执行的周期
> this.period = 0;
> //保存任务被添加到ScheduledFutureTask中的序号
> this.sequenceNumber = sequencer.getAndIncrement();
> ```
>
> 在schedule方法中，创建完ScheduledFutureTask对象后，回调用delayedExecture方法，执行任务。

#### 3.10、delayedExecute方法

> ```java
> private void delayedExecute(RunnableScheduledFuture<?> task) {
>     if (isShutdown()) //如果线程池关闭，使用拒绝策略
>         reject(task); 
>     else {
>         super.getQueue().add(task); //添加到阻塞队列中
>         if (isShutdown() &&
>             !canRunInCurrentRunState(task.isPeriodic()) &&
>             remove(task))
>             task.cancel(false);
>         else
>             ensurePrestart(); //确保线程池中至少有一个线程启动，即使corePoolSize为0，该方法在ThreadPoolExecutor中实现
>     }
> }
> ```
>
> 解释下这两个if判断：
>
> >  1、 如果不是SHUTDOWN的状态，执行else，否则执行步骤2。
> >
> > 2、 如果在当前线程池运行状态下可以执行任务，执行else，否则执行步骤3。
> >
> > 3、 从阻塞队列中删除任务，如果失败，执行else，否则执行步骤四。
> >
> > 4、 取消任务，但不中断执行中的任务。

> 对于步骤2，可以通过setContinueExistingPeriodicTasksAfterShutdownPolicy方法设置在线程池关闭时，周期任务继续执行，默认为false，也就是在线程池关闭时，不在执行周期任务。
>
> ensurePrestart方法在ThreadPoolExecutor中的定义
>
> ```java 
> void ensurePrestart() {
>     int wc = workerCountOf(ctl.get());
>     if (wc < corePoolSize)
>         addWorker(null, true);
>     else if (wc == 0)
>         addWorker(null, false);
> }
> ```
>
> 调用了addWorker方法，可以在[深入理解Java线程池：ThreadPoolExecutor](http://www.ideabuffer.cn/2017/04/04/深入理解Java线程池：ThreadPoolExecutor/#addWorker方法)中查看addWorker方法的介绍，线程池中的工作线程是通过该方法来启动并执行任务的。

#### 3.11、ScheduledFutureTask中的run方法

> 回顾一下线程池的执行过程：当线程池中的工作线程执行启动时，不断的从阻塞队列中取出任务并执行，当然，取出的任务实现了Runnable接口，所以是通过调用任务的run方法来执行任务的。
>
> 这里的任务类型是ScheduledFutureTask，所以看一下ScheduledFutureTask中的run方法。
>
> ```java
> public void run() {
>     boolean periodic = isPeriodic(); //是否是周期任务
>     if (!canRunInCurrentRunState(periodic)) //如果当前线程池运行状态下不可以执行任务，取消该任务
>         cancel(false);
>     else if (!periodic) //如果不是周期性任务，调用FutureTask的run方法执行。
>         ScheduledFutureTask.super.run();
>    	//如果是周期性任务，调用FutureTask中的tunAndReset方法执行，
>     //runAndReset方法不会设置执行结果，所以可以重复执行任务。
>     else if (ScheduledFutureTask.super.runAndReset()) {
>       	//计算下次执行该任务的时间
>         setNextRunTime();
>       	//重复执行任务
>         reExecutePeriodic(outerTask);
>     }
> }
> ```
>
> 有关FutureTask的run方法和runAndReset方法，可以参考[FutureTask源码解析](http://www.ideabuffer.cn/2017/04/06/FutureTask源码解析/FutureTask源码解析/)。
>
> 分析下执行过程：
>
> > 1、如果当前线程池运行状态不可以执行任务，取消任务，直接返回，否则执行步骤二。
> >
> > 2、如果不是周期性任务，则直接调用FutureTask的run方法，设置执行结果，然后直接返回，否则执行步骤三。
> >
> > 3、如果是周期性任务，则调用FutureTask的runAndReset方法，不会设置执行结果，然后直接返回，然后执行步骤四和步骤五。
> >
> > 4、计算下次执行该任务的时间。
> >
> > 5、重复执行任务。

#### 3.12、ScheduledFutureTask的reExecutePeriodic方法

> ```java
> void reExecutePeriodic(RunnableScheduledFuture<?> task) {
>     if (canRunInCurrentRunState(true)) {
>         super.getQueue().add(task);
>         if (!canRunInCurrentRunState(true) && remove(task))
>             task.cancel(false);
>         else
>             ensurePrestart();
>     }
> }
> ```
>
> 该方法和delayedExecute方法类似，不同的是：
>
> > 1. 由于调用reExecutePeriodic方法时,已经执行过一次周期性任务了，所以不会reject当前任务；
> > 2. 传入的任务一定是周期性任务。

#### 3.13、onShutdown方法

> onShutdown方法是ThreadPoolExecutor中的钩子方法，在ThreadPoolExecutor中什么都没有做，参考[深入理解Java线程池：ThreadPoolExecutor](http://www.ideabuffer.cn/2017/04/04/深入理解Java线程池：ThreadPoolExecutor/)，该方法是在执行shutdown方法时被调用：
>
> ```java
> @Override void onShutdown() {
>     BlockingQueue<Runnable> q = super.getQueue();
>   	//获取在线程池中shutdown的情况下，是否继续执行现有的延迟任务
>     boolean keepDelayed =
>         getExecuteExistingDelayedTasksAfterShutdownPolicy();
>   	//获取线程池中shutdown的情况下，是否继续执行现有定期任务
>     boolean keepPeriodic =
>         getContinueExistingPeriodicTasksAfterShutdownPolicy();
>   	//如果在线程池shutdown的情况下，不继续执行现有的延迟任务和定期任务
>   	//则依次取消任务，否则则根据取消状态来判断
>     if (!keepDelayed && !keepPeriodic) {
>         for (Object e : q.toArray())
>             if (e instanceof RunnableScheduledFuture<?>)
>                 ((RunnableScheduledFuture<?>) e).cancel(false);
>         q.clear();
>     }
>     else {
>         // Traverse snapshot to avoid iterator exceptions
>         for (Object e : q.toArray()) {
>             if (e instanceof RunnableScheduledFuture) {
>                 RunnableScheduledFuture<?> t =
>                     (RunnableScheduledFuture<?>)e;
>               	//如果在shutdown状态下不继续的延迟任务或者定期任务，则从队列中删除，并取消任务。
>                 if ((t.isPeriodic() ? !keepPeriodic : !keepDelayed) ||
>                     t.isCancelled()) { // also remove if already cancelled
>                     if (q.remove(t))
>                         t.cancel(false);
>                 }
>             }
>         }
>     }
>     tryTerminate();
> }
> ```

### 4、DelayedWorkQueue

> ScheduledThreadPoolExecutor之所以自己实现阻塞的工作队列，是因为ScheduledThreadPoolExecutor要求的工作队列有些特殊：
>
> DelayedWorkQueue是基于堆的数据结构、数组的存储结构。类似于DelayQueue和PriorityQueue，在执行定时任务的时候，每个任务的执行时间都不相同，所以DelayedWorkQueue的工作就是按照执行时间的升序进行排列的，执行时间越近的任务排在队列的前面。
>
> （注意，这里的顺序并不是绝对的，堆中的排序只保证了子节点的下次执行时间要比父节点的下次执行时间要大，而叶子节点之间并不一定是有顺序的。）
>
> 堆结构如下图所示：
>
> ![image-20200903151011870](/Users/peng/Library/Application Support/typora-user-images/image-20200903151011870.png)
>
> 可见DelayWorkQueue是基于最小堆结构的队列。堆结构可以使用数组表示，可以转换成如下的数组：
>
> ![image-20200903151222477](/Users/peng/Library/Application Support/typora-user-images/image-20200903151222477.png)
>
> 在这种结构中可以发现有如下特性：
>
> 假设索引值从0开始，子节点的索引值为K，父节点的索引值为P，则：
>
> > 1、一个节点的左子节点的索引为：K = P * 2 + 1；
> >
> > 2、一个节点的右子节点的索引为：K = （ P + 1）* 2；
> >
> > 3、一个节点的父节点的索引为：P =（K - 1）/ 2；

#### 4.1、为什么使用DelayedWorkQueue呢？

> 定时任务执行时，需要取出最近要执行的任务，所以任务在队列中每次出队列时一定要是当前队列中执行时间要靠前的，所以自然要使用优先级队列。
>
> DelayedWorkQueue是一个优先级队列，他可以保证每次出队的任务都是当前队列中执行时间最靠前的，由于它是基于堆结构的队列，堆结构在执行插入操作或者删除操作，它的时间复杂度为O(logN)。

#### 4.2、DelayedWorkQueue的属性

> ```java
> //队列初始容量
> private static final int INITIAL_CAPACITY = 16;
> //根据初始容量创建RunnableScheduledFuture类型的数组
> private RunnableScheduledFuture<?>[] queue =
>     new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
> private final ReentrantLock lock = new ReentrantLock();
> private int size = 0;
> //leader线程
> private Thread leader = null;
> //当较新的任务在队列头部可用时，或者新线程在队列中需要称为leader，则通过该条件发出信号。
> private final Condition available = lock.newCondition();
> ```
>
> 注意这里的leader，它是leader-follower模式的变体，用于减少不必要的定时等待，什么意思呢？对于多线程的网络模型来说：
>
> > 所有线程会有三种身份中的一种：leader和follower，以及一个干活中的状态：proccesser。它的基本原则就是，永远最多只有一个leader。而所有follower都在等待成为leader。线程池启动时会自动产生一个Leader负责等待网络IO事件，当有一个事件产生时，Leader线程首先通知一个Follower线程将其提拔为新的Leader，然后自己就去干活了，去处理这个网络事件，处理完毕后加入Follower线程等待队列，等待下次成为Leader。这种方法可以增强CPU高速缓存相似性，及消除动态内存分配和线程间的数据交换。
>
> *参考自：http://blog.csdn.net/goldlevi/article/details/7705180*
>
> 具体leader的作用在分析take方法时再详细介绍。

#### 4.3、offer方法

> 既然是阻塞队列，入队的操作如add和put方法都调用了offer方法，下面查看一下offer方法：
>
> ```java
> public boolean offer(Runnable x) {
>     if (x == null)
>         throw new NullPointerException();
>     RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
>     final ReentrantLock lock = this.lock;
>     lock.lock();
>     try {
>         int i = size;
>       	//queue是一个RunnableScheduledFuture类型的数组，容量不够，需要扩容。
>         if (i >= queue.length)
>             grow();
>         size = i + 1;
>       	//i==0说明队列中还没有数据
>         if (i == 0) {
>             queue[0] = e;
>             setIndex(e, 0);
>         //当i！=0，需要对堆进行重新排序
>         } else {
>             siftUp(i, e);
>         }
>       	//如果传入的任务已经是队列的第一个节点了，这时available需要发出信号
>         if (queue[0] == e) {
>           	//leader设置为null，为了使take方法中的线程在通过available.signal();后会执行available.awaitNanos(delay);
>             leader = null;
>             available.signal();
>         }
>     } finally {
>         lock.unlock();
>     }
>     return true;
> }
> ```
>
> 有关Condition的介绍请参考[深入理解AbstractQueuedSynchronizer（三）](http://www.ideabuffer.cn/2017/03/20/深入理解AbstractQueuedSynchronizer（三）/)
>
> 这里的重点是siftUp方法

#### 4.4、siftUp方法

> ```java
> private void siftUp(int k, RunnableScheduledFuture<?> key) {
>     while (k > 0) {
>       	//找到父节点的索引
>         int parent = (k - 1) >>> 1;
>       	//获取父节点
>         RunnableScheduledFuture<?> e = queue[parent];
>       	//如果key节点的执行时间大于父节点的执行时间，则不需要排序
>         if (key.compareTo(e) >= 0)
>             break;
>       	//如果key.conpareTo(e) < 0,说明key节点的执行时间小于父节点的执行时间，则将父子节点交换位置
>         queue[k] = e;
>       	//设置索引为k
>         setIndex(e, k);
>         k = parent;
>     }
>   	//key设置为排序后的位置中
>     queue[k] = key;
>     setIndex(key, k);
> }
> ```
>
> 代码很好理解，就是循环的根据key节点与它的父节点来判断，如果key节点的执行时间小于父节点，则将两个节点交换，使执行时间靠前的节点排列在队列的前面。
>
> 假设新入队的节点的延迟时间（调用getDelay()方法获得）是5，执行过程如下：
>
> > **1、先将新的节点添加到数组的尾部，这时新节点的索引k为7：**
> >
> > ![image-20200903154648437](/Users/peng/Library/Application Support/typora-user-images/image-20200903154648437.png)
> >
> > **2、计算新父节点的索引：parent = (k - 1) >>> 1，parent = 3，那么queue[3]的时间间隔值为8，因为 5 < 8 ，将执行queue[7] = queue[3]：**
> >
> > ![image-20200903154843414](/Users/peng/Library/Application Support/typora-user-images/image-20200903154843414.png)
> >
> > **3、这时将k设置为3，继续循环，再次计算parent为1，queue[1]的时间间隔为3，因为 5 > 3 ，这时退出循环，最终k为3：**
> >
> > ![image-20200903154922980](/Users/peng/Library/Application Support/typora-user-images/image-20200903154922980.png)
>
> 可见，每次新增节点时，只是根据父节点来判断，而不会影响兄弟节点。
>
> 另外，setIndex方法只是设置了ScheduledFutureTask中的heapIndex属性：
>
> ```java
> private void setIndex(RunnableScheduledFuture<?> f, int idx) {
>     if (f instanceof ScheduledFutureTask)
>         ((ScheduledFutureTask)f).heapIndex = idx;
> }
> ```

#### 4.5 、take方法

> ```java
> public RunnableScheduledFuture<?> take() throws InterruptedException {
>     final ReentrantLock lock = this.lock;
>     lock.lockInterruptibly();
>     try {
>         for (;;) {
>             RunnableScheduledFuture<?> first = queue[0];
>             if (first == null)
>                 available.await();
>             else {
>               	//计算当前时间到执行时间到时间间隔
>                 long delay = first.getDelay(NANOSECONDS);
>                 if (delay <= 0)
>                     return finishPoll(first);
>                 first = null; // don't retain ref while waiting
>               	//leader不为空，阻塞线程
>                 if (leader != null)
>                     available.await();
>                 else {
>                   	//leader为空，则把当前线程设置为leader
>                     Thread thisThread = Thread.currentThread();
>                     leader = thisThread;
>                     try {
>                       	//阻塞到执行时间
>                         available.awaitNanos(delay);
>                     } finally {
>                         if (leader == thisThread)
>                           	//设置leader为null，让其他线程执行available.awaitNanos(delay);
>                             leader = null;
>                     }
>                 }
>             }
>         }
>     } finally {
>       	//如果leader不为空，则说明leader线程正在执行available.awaitNanos(delay);
>       	//如果queue[0] == null，则说明队列为空 
>         if (leader == null && queue[0] != null)
>             available.signal();
>         lock.unlock();
>     }
> }
> ```
>
> take方法是什么时候调用的呢？在[深入理解Java线程池：ThreadPoolExecutor](http://www.ideabuffer.cn/2017/04/04/深入理解Java线程池：ThreadPoolExecutor/)中，介绍了getTask方法，工作线程会循环地从workQueue中取任务。但定时任务却不同，因为如果一旦getTask方法取出了任务就开始执行了，而这时可能还没有到执行的时间，所以在take方法中，要保证只有在到指定的执行时间的时候任务才可以被取走。
>
> 再来说一下leader的作用，这里的leader是为了减少不必要的定时等待，当一个线程成为leader时，它只等待下一个节点的时间间隔，但其它线程无限期等待。 leader线程必须在从take（）或poll（）返回之前signal其它线程，除非其他线程成为了leader。
>
> 举例来说，如果没有leader，那么在执行take时，都要执行`available.awaitNanos(delay)`，假设当前线程执行了该段代码，这时还没有signal，第二个线程也执行了该段代码，则第二个线程也要被阻塞。多个这时执行该段代码是没有作用的，因为只能有一个线程会从take中返回queue[0]（因为有lock），其他线程这时再返回for循环执行时取的queue[0]，已经不是之前的queue[0]了，然后又要继续阻塞。
>
> 所以，为了不让多个线程频繁的做无用的定时等待，这里增加了leader，如果leader不为空，则说明队列中第一个节点已经在等待出队，这时其它的线程会一直阻塞，减少了无用的阻塞（注意，在finally中调用了signal()来唤醒一个线程，而不是signalAll()）。

#### 4.6、poll方法

> 下面看下poll方法，与take类似，但这里要提供超时功能：
>
> ```java
> public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)
>     throws InterruptedException {
>     long nanos = unit.toNanos(timeout);
>     final ReentrantLock lock = this.lock;
>     lock.lockInterruptibly();
>     try {
>         for (;;) {
>             RunnableScheduledFuture<?> first = queue[0];
>             if (first == null) {
>                 if (nanos <= 0)
>                     return null;
>                 else
>                     nanos = available.awaitNanos(nanos);
>             } else {
>                 long delay = first.getDelay(NANOSECONDS);
>               	// 如果delay <= 0，说明已经到了任务执行的时间，返回。
>                 if (delay <= 0)
>                     return finishPoll(first);
> 	              // 如果nanos <= 0，说明已经超时，返回null
>                 if (nanos <= 0)
>                     return null;
>                 first = null; // don't retain ref while waiting
>               	// nanos < delay 说明需要等待的时间小于任务要执行的延迟时间
>                 // leader != null 说明有其它线程正在对任务进行阻塞
>                 // 这时阻塞当前线程nanos纳秒
>                 if (nanos < delay || leader != null)
>                     nanos = available.awaitNanos(nanos);
>                 else {
>                     Thread thisThread = Thread.currentThread();
>                     leader = thisThread;
>                     try {
> 	                      // 这里的timeLeft表示delay减去实际的等待时间
>                         long timeLeft = available.awaitNanos(delay);
> 	                      // 计算剩余的等待时间
>                         nanos -= delay - timeLeft;
>                     } finally {
>                         if (leader == thisThread)
>                             leader = null;
>                     }
>                 }
>             }
>         }
>     } finally {
>         if (leader == null && queue[0] != null)
>             available.signal();
>         lock.unlock();
>     }
> }
> ```

#### 4.7、finishPoll方法

> 当调用了take或者poll方法能够获取到任务时，会调用该方法进行返回
>
> ```java
> private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
>     // 数组长度-1
>     int s = --size;
>     // 取出最后一个节点
>     RunnableScheduledFuture<?> x = queue[s];
>     queue[s] = null;
>     // 长度不为0，则从第一个元素开始排序，目的是要把最后一个节点放到合适的位置上
>     if (s != 0)
>         siftDown(0, x);
>     setIndex(f, -1);
>     return f;
> }
> ```

#### 4.8、siftDown方法

> siftDown方法使堆从k开始向下调整：
>
> ```Java
> private void siftDown(int k, RunnableScheduledFuture<?> key) {
>     // 根据二叉树的特性，数组长度除以2，表示取有子节点的索引
>     int half = size >>> 1;
>     // 判断索引为k的节点是否有子节点
>     while (k < half) {
>         // 左子节点的索引
>         int child = (k << 1) + 1;
>         RunnableScheduledFuture<?> c = queue[child];
>         // 右子节点的索引
>         int right = child + 1;
>         // 如果有右子节点并且左子节点的时间间隔大于右子节点，取时间间隔最小的节点
>         if (right < size && c.compareTo(queue[right]) > 0)
>             c = queue[child = right];
>         // 如果key的时间间隔小于等于c的时间间隔，跳出循环
>         if (key.compareTo(c) <= 0)
>             break;
>         // 设置要移除索引的节点为其子节点
>         queue[k] = c;
>         setIndex(c, k);
>         k = child;
>     }
>     // 将key放入索引为k的位置
>     queue[k] = key;
>     setIndex(key, k);
> }
> ```
>
> siftDown方法执行时包含两种情况，一种是没有子节点，一种是有子节点（根据half判断）。例如：
>
> **没有子节点的情况：**
>
> 假设初始的堆如下：
>
> ![image-20200903173946930](/Users/peng/Library/Application Support/typora-user-images/image-20200903173946930.png)
>
> 假设 k = 3 ，那么 k = half ，没有子节点，在执行siftDown方法时直接把索引为3的节点设置为数组的最后一个节点：
>
> ![image-20200903174008605](/Users/peng/Library/Application Support/typora-user-images/image-20200903174008605.png)
>
> **有子节点的情况：**
>
> 假设 k = 0 ，那么执行以下步骤：
>
> > 1. 获取左子节点，child = 1 ，获取右子节点， right = 2 ：
> >
> >    ![image-20200903174109030](/Users/peng/Library/Application Support/typora-user-images/image-20200903174109030.png)
> >
> > 2. 由于 `right < size` ，这时比较左子节点和右子节点时间间隔的大小，这里 3 < 7 ，所以 c = queue[child] ；
> >
> > 3. 比较key的时间间隔是否小于c的时间间隔，这里不满足，继续执行，把索引为k的节点设置为c，然后将k设置为child，；
> >
> >    ![image-20200903174159317](/Users/peng/Library/Application Support/typora-user-images/image-20200903174159317.png)
> >
> > 4. 因为 half = 3 ，k = 1 ，继续执行循环，这时的索引变为：
> >
> >    ![image-20200903174240048](/Users/peng/Library/Application Support/typora-user-images/image-20200903174240048.png)
> >
> > 5. 这时再经过如上判断后，将k的值为3，最终的结果如下：
> >
> >    ![image-20200903174309107](/Users/peng/Library/Application Support/typora-user-images/image-20200903174309107.png)
> >
> > 6. 最后，如果在finishPoll方法中调用的话，会把索引为0的节点的索引设置为-1，表示已经删除了该节点，并且size也减了1，最后的结果如下：
> >
> >    ![image-20200903174336870](/Users/peng/Library/Application Support/typora-user-images/image-20200903174336870.png)
>
> 可见，siftdown方法在执行完并不是有序的，但可以发现，子节点的下次执行时间一定比父节点的下次执行时间要大，由于每次都会取左子节点和右子节点中下次执行时间最小的节点，所以还是可以保证在take和poll时出队是有序的。

#### 4.9、remove方法

> ```java
> public boolean remove(Object x) {
>     final ReentrantLock lock = this.lock;
>     lock.lock();
>     try {
>         int i = indexOf(x);
>         if (i < 0)
>             return false;
> 
>         setIndex(queue[i], -1);
>         int s = --size;
>         RunnableScheduledFuture<?> replacement = queue[s];
>         queue[s] = null;
>         if (s != i) {
>             // 从i开始向下调整
>             siftDown(i, replacement);
>             // 如果queue[i] == replacement，说明i是叶子节点
>             // 如果是这种情况，不能保证子节点的下次执行时间比父节点的大
>             // 这时需要进行一次向上调整
>             if (queue[i] == replacement)
>                 siftUp(i, replacement);
>         }
>         return true;
>     } finally {
>         lock.unlock();
>     }
> }
> ```
>
> 假设初始的堆结构如下：
>
> ![image-20200903174500070](/Users/peng/Library/Application Support/typora-user-images/image-20200903174500070.png)
>
> 这时要删除8的节点，那么这时 k = 1，key为最后一个节点：
>
> ![image-20200903174518582](/Users/peng/Library/Application Support/typora-user-images/image-20200903174518582.png)
>
> 这时通过上文对siftDown方法的分析，siftDown方法执行后的结果如下：
>
> ![image-20200903174550729](/Users/peng/Library/Application Support/typora-user-images/image-20200903174550729.png)
>
> 这时会发现，最后一个节点的值比父节点还要小，所以这里要执行一次siftUp方法来保证子节点的下次执行时间要比父节点的大，所以最终结果如下：
>
> ![image-20200903174610842](/Users/peng/Library/Application Support/typora-user-images/image-20200903174610842.png)

### 5、总结

> 本文详细分析了ScheduedThreadPoolExecutor的实现，主要介绍了以下方面：
>
> - 与Timer执行定时任务的比较，相比Timer，ScheduedThreadPoolExecutor有什么优点；
> - ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，所以它也是一个线程池，也有coorPoolSize和workQueue，ScheduledThreadPoolExecutor特殊的地方在于，自己实现了优先工作队列DelayedWorkQueue；
> - ScheduedThreadPoolExecutor实现了ScheduledExecutorService，所以就有了任务调度的方法，如schedule，scheduleAtFixedRate和scheduleWithFixedDelay，同时注意他们之间的区别；
> - 内部类ScheduledFutureTask继承自FutureTask，实现了任务的异步执行并且可以获取返回结果。同时也实现了Delayed接口，可以通过getDelay方法获取将要执行的时间间隔；
> - 周期任务的执行其实是调用了FutureTask类中的runAndReset方法，每次执行完不设置结果和状态。参考[FutureTask源码解析](http://www.ideabuffer.cn/2017/04/06/FutureTask源码解析/)；
> - 详细分析了DelayedWorkQueue的数据结构，它是一个基于最小堆结构的优先队列，并且每次出队时能够保证取出的任务是当前队列中下次执行时间最小的任务。同时注意一下优先队列中堆的顺序，堆中的顺序并不是绝对的，但要保证子节点的值要比父节点的值要大，这样就不会影响出队的顺序。
>
> 总体来说，ScheduedThreadPoolExecutor的重点是要理解下次执行时间的计算，以及优先队列的出队、入队和删除的过程，这两个是理解ScheduedThreadPoolExecutor的关键。



































