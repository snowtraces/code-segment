# 计数器定时持久化

+ [原文](http://ifeve.com/非阻塞同步算法实战（四）-计数器定时持久化/)

## 问题背景及要求
+ 需要对评论进行点赞次数和被评论次数进行统计，或者更多维度
+ 要求高并发、高性能计数，允许极端情况丢失一些统计次数，例如宕机
+ 评论很多，不能为每一个评论都一直保留其计数器，计数器需要有回收机制

## 问题抽象及分析
根据以上需求，为了方便编码与测试，我们把需求转化为以下接口
```java
/**
 * 计数器
 */
public interface Counter {
    /**
     * 取出统计数据，用Saver去持久化(仅定时器会调用，无并发)
     *
     * @param saver
     */
    void save(Saver saver);

    /**
     * 计数(有并发)
     *
     * @param key     业务ID
     * @param like    点赞
     * @param comment 评论
     */
    void add(String key, int like, int comment);

    /**
     * 持久化器，将数量持久化到数据库等
     */
    @FunctionalInterface
    interface Saver {
        void save(String key, int like, int comment);
    }
}

```
简单分析可知，计数器比较简单，用AtomicInteger便能保证原子性，但考虑到计数器会被回收，则可能会出现这样的场景：某计数器已被回收了，此时继续在该计数器上计数，便会造成数据丢失，因此要处理该并发问题

## 解决方案
### 方案一
使用原生锁来解决竞争问题
```java
/**
 * 直接对所有操作上锁，来保证线程安全
 */
public class SynchronizedCounter implements Counter {
    private HashMap<String, Adder> map = new HashMap<>();

    @Override
    public synchronized void save(Saver saver) {
        map.forEach((key, value) -> {//因为已加锁，所以可以安全地取数据
            saver.save(key, value.like, value.comment);
        });
        map = new HashMap<>();
    }

    @Override
    public synchronized void add(String key, int like, int comment) {
        //因为已加锁，所以可以安全地更新数据
        Adder adder = map.computeIfAbsent(key, x -> new Adder());
        adder.like += like;
        adder.comment += comment;
    }

    static class Adder {
        private int like;
        private int comment;
    }
}
```
方案点评：该方案让业务线程和定时保存线程竞争同一把实例锁，让他们互斥地访问，解决了竞争问题，但锁粒度太粗爆，性能低下

### 方案二
为了循序渐进，我们把“计数器需要有回收机制”这条要求去掉，这样我们可以很容易地利用上AtomicInteger这个类
```java
/**
 * 不回收计数器，问题变得简单许多
 */
public class IncompleteCounter implements Counter {
    private ConcurrentHashMap<String, Adder> map = new ConcurrentHashMap<>();
    @Override
    public void save(Saver saver) {
        map.forEach((key, value)->{//利用了AtomicInteger的原子特性，可以线程安全地取出所有计数，并置0(因为还会继续使用)
            saver.save(key, value.like.getAndSet(0), value.comment.getAndSet(0));
        });
        //因为不回收，所以不用考虑Adder被回收丢弃后，仍被其它线程使用的情况(因为没有锁，所以这种情况是可能发生的)
    }

    @Override
    public void add(String key, int like, int comment) {
        Adder adder = map.computeIfAbsent(key, k -> new Adder());
        adder.like.addAndGet(like);//利用AtomicInteger的原子特性，保证了线程安全
        adder.comment.addAndGet(comment);
    }
    static class Adder{
        AtomicInteger like = new AtomicInteger();
        AtomicInteger comment = new AtomicInteger();
    }
}
```
方案点评：除了没解决回收问题，简单高效

### 方案三
因为调用save的线程没有并发情况，阻塞也没关系，经分析可巧妙地使用读写锁，同时又不让add方法进入阻塞

```java
/**
 * 巧妙地利用读写锁，及save方法可阻塞的特点，实现add操作无阻塞
 */
public class ReadWriteLockCounter implements Counter {
    private volatile MapWithLock mapWithLock = new MapWithLock();

    @Override
    public void save(Saver saver) {
        MapWithLock preMapWithLock = mapWithLock;
        mapWithLock = new MapWithLock();
        //不会一直阻塞，因为mapWithLock已被替换，新的add调用会拿到新的mapWithLock
        preMapWithLock.lock.writeLock().lock();
        preMapWithLock.map.forEach((key,value)->{
            //value已经废弃，故无需value.like.getAndSet(0)
            saver.save(key, value.like.get(), value.comment.get());
        });
        //不能释放该锁，否则add方法中，对被替换掉的MapWithLock.lock执行tryLock会成功
        //也许，这是你第一次见到的不需要且不允许释放的锁:)
    }

    @Override
    public void add(String key, int like, int comment) {
        MapWithLock mapWithLock;
        //如果通过tryLock获取锁失败，则表示该mapWithLock已经被废弃了（因为只有废弃了的MapWithLock才会加写锁），故重新获取最新的mapWithLock
        while(!(mapWithLock = this.mapWithLock).lock.readLock().tryLock());
        try{
            Adder adder = mapWithLock.map.computeIfAbsent(key, k -> new Adder());
            adder.like.getAndAdd(like);
            adder.comment.getAndAdd(comment);
        }finally {
            mapWithLock.lock.readLock().unlock();
        }
    }

    static class Adder{
        private AtomicInteger like = new AtomicInteger();
        private AtomicInteger comment = new AtomicInteger();

    }
    static class MapWithLock{
        private ConcurrentHashMap<String, Adder> map = new ConcurrentHashMap<>();
        private ReadWriteLock lock = new ReentrantReadWriteLock();
    }
}
```
方案点评：减少了锁的粒度，同时add线程可以相互兼容，大幅提升了并发能力，save线程虽会阻塞，但结合其定时执行的特点，并不受影响，且即使极端情况也不会一直阻塞

### 方案四
使用一个原子的state来替换LockCounter中的ReadWriteLock(因为只使用到了它的部分特性)，实现wait-free，获得更高性能

```java
/**
 * ReadWriteLockCounter的改进版，去掉ReadWriteLock，结合当前场景，实现一个wait-free的简易读写锁<br/>
 */
public class CustomLockCounter implements Counter {
    private volatile MapWithState mapWithState = new MapWithState();

    @Override
    public void save(Saver saver) {
        MapWithState preMapWithState = mapWithState;
        mapWithState = new MapWithState();
        //compareAndSet失败则表示该MapWithState正在被使用，等其使用完，它不会一直失败，因为mapWithState已经被替换
        while(!preMapWithState.state.compareAndSet(0,Integer.MIN_VALUE)){
            Thread.yield();
        }
        preMapWithState.map.forEach((key, value)->{
            //value已经废弃，故无需value.like.getAndSet(0)
            saver.save(key, value.like.get(), value.comment.get());
        });
    }

    @Override
    public void add(String key, int like, int comment) {
        MapWithState mapWithState;//add的并发，不可能将Integer.MIN_VALUE自增成正数(设置为Integer.MIN_VALUE时，该MapWithState已经被废弃了)
        while((mapWithState = this.mapWithState).state.getAndIncrement()<0);
        try{
            Adder adder = mapWithState.map.computeIfAbsent(key, k -> new Adder());
            adder.like.getAndAdd(like);
            adder.comment.getAndAdd(comment);
        }finally {
            mapWithState.state.getAndDecrement();
        }
    }

    static class Adder{
        private AtomicInteger like = new AtomicInteger();
        private AtomicInteger comment = new AtomicInteger();

    }
    static class MapWithState {
        private ConcurrentHashMap<String, Adder> map = new ConcurrentHashMap<>();
        private AtomicInteger state = new AtomicInteger();
    }
}
```
方案点评：保留了前一方案ReadWriteLockCounter的优点，同时结合场景的特点做了些优化，本质就是将CAS失败重试循环替换成了一条fetch-and-add指令，如果不是因为save是低频执行，本方案可能是最高效的了(暂且忽略ConcurrentHashMap等其它可能的优化空间)

### 方案五
先假定不会发生竞争，然后检测竞争情况，如果发生竞争，则补偿

```java
/**
 * 乐观地假定不会发生竞争，如果发生了，则尝试进行补偿
 */
public class CompensationCounter implements Counter {
    private ConcurrentHashMap<String, Adder> map = new ConcurrentHashMap<>();
    @Override
    public void save(Saver saver) {
        for(Iterator<Map.Entry<String, Adder>> it = map.entrySet().iterator(); it.hasNext();){
            Map.Entry<String, Adder> entry = it.next();
            it.remove();
            entry.getValue().discarded = true;
            saver.save(entry.getKey(), entry.getValue().like.getAndSet(0), entry.getValue().comment.getAndSet(0));//需将计数器置0，此处存在竞争
        }
    }

    @Override
    public void add(String key, int like, int comment) {
        Adder adder = map.computeIfAbsent(key, k -> new Adder());
        adder.like.addAndGet(like);
        adder.comment.addAndGet(comment);
        if(adder.discarded){//如果数量加在了废弃的Adder上面，则执行补偿逻辑
            int likeTemp = adder.like.getAndSet(0);
            int commentTemp = adder.comment.getAndSet(0);
            //即使此后又有线程在计数器上计数了也无妨
            if(likeTemp != 0 || commentTemp != 0){
                add(key, likeTemp, commentTemp);//补偿
            }//也可能已经被其它线程取走了，但并不影响业务正确性
        }
    }
    static class Adder{
        AtomicInteger like = new AtomicInteger();
        AtomicInteger comment = new AtomicInteger();
        volatile boolean discarded = false;//只有保存线程会将它改为true，故使用volatile便能保证线程安全
    }
}
```
方案点评：跟乐观锁的思路类似，在竞争激烈的情况下，一般不会有最优性能，但此处因为save方法是低频执行的且自身无并发，add方法才有高并发，故失败补偿其实很少真正被执行，这也是为什么测试结果中本方案性能最优的原因

## 性能测试
最终我们来测试一下各方案的性能，因为我们抽象出了一个统一的接口，故测试也较为容易

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;

public class CounterTester {
    private static final int THREAD_SIZE = 6;//add方法的并发线程数
    private static final int ADD_SIZE = 5000000;//测试规模
    private static final int KEYS_SIZE = 128*1024;
    public static void main(String[] args) throws InterruptedException {
        Counter[] counters = new Counter[]{new SynchronizedCounter(), new IncompleteCounter(), new ReadWriteLockCounter(), new CustomLockCounter(), new CompensationCounter()};
        String[] keys = new String[KEYS_SIZE];
        Random random = new Random();
        for (int i = 0; i < keys.length; i++) {
            keys[i]=String.valueOf(random.nextInt(KEYS_SIZE*1024));
        }
        for (Counter counter : counters) {
            AtomicInteger totalLike = new AtomicInteger();
            AtomicInteger totalComment = new AtomicInteger();
            AtomicInteger savedTotalLike = new AtomicInteger();
            AtomicInteger savedTotalComment = new AtomicInteger();
            Counter.Saver saver = (key, like, comment) -> {
                savedTotalLike.addAndGet(like);//模拟被持久化到数据库，记录数量以便后续校验正确性
                savedTotalComment.addAndGet(comment);//同上
            };
            CountDownLatch latch = new CountDownLatch(THREAD_SIZE);
            long start = System.currentTimeMillis();
            for (int i = 0; i < THREAD_SIZE; i++) {
                new Thread(()->{
                    Random r = new Random();
                    int like, comment;
                    for (int j = 0; j < ADD_SIZE; j++) {
                        like = 2;
                        comment = 4;
                        counter.add(keys[r.nextInt(KEYS_SIZE)], like, comment);
                        totalLike.addAndGet(like);
                        totalComment.addAndGet(comment);
                    }
                    latch.countDown();
                }).start();
            }
            Thread saveThread = new Thread(()->{
                while(latch.getCount() != 0){
                    try {
                        Thread.sleep(100);//模拟100毫秒执行一次持久化
                    } catch (InterruptedException e) {}
                    counter.save(saver);
                }
                counter.save(saver);

            });
            saveThread.start();
            latch.await();
            System.out.println(counter.getClass().getSimpleName() +" cost:\t"+(System.currentTimeMillis() - start));
            saveThread.join();
            boolean error = savedTotalLike.get() != totalLike.get() || savedTotalComment.get() != totalComment.get();
            (error?System.err:System.out).println("saved:\tlike="+savedTotalLike.get()+"\tcomment="+savedTotalComment.get());
            (error?System.err:System.out).println("added:\tlike="+totalLike.get()+"\tcomment="+totalComment.get()+"\n");
        }
    }
}
```
在jdk11(jdk8也基本一致)下的测试结果如下：

> 注：方案二的IncompleteCounter并未完成回收，仅作对比

```log
SynchronizedCounter cost:	12377
saved:	like=60000000	comment=120000000
added:	like=60000000	comment=120000000

IncompleteCounter cost:	2560
saved:	like=60000000	comment=120000000
added:	like=60000000	comment=120000000

ReadWriteLockCounter cost:	7902
saved:	like=60000000	comment=120000000
added:	like=60000000	comment=120000000

CustomLockCounter cost:	3541
saved:	like=60000000	comment=120000000
added:	like=60000000	comment=120000000

CompensationCounter cost:	2093
saved:	like=60000000	comment=120000000
added:	like=60000000	comment=120000000
```

## 小结
非阻塞同步算法一般不需要我们去设计，直接使用现有的工具便可，但如果真想通过它进一步去压榨性能，应细心分析各线程穿插执行的情况，同时结合业务场景来考虑(也许在A场景不允许的情况，在B场景是允许的)