# ThreadLocal

- ```java
  public void set(T value) {
      Thread t = Thread.currentThread();
      ThreadLocalMap map = getMap(t);
      if (map != null) {
          map.set(this, value);
      } else {
          createMap(t, value);
      }
  }
  
  void createMap(Thread t, T firstValue) {
      t.threadLocals = new ThreadLocalMap(this, firstValue);
  }
  ```

  - this作为key不是Thread，而是ThreadLocal
  - new了一个ThreadLocalMap，ThreadLocal为key，由ThreadLocal塞到当前Thread的threadLocals中
  - 多个Thread一个ThreadLocal：不同的Thread其map也不同，不会混淆其中内容
  - 一个Thread多个ThreadLocal：一个Thread只能有一个ThreadLocalMap，第一个ThreadLocal创建并存入其中，之后的ThreadLocal将自己作为key存入该map中
  - 多个Thread多个ThreadLocal：每个Thread遇到不同的ThreadLocal后都会将其作为key存入或者取出

- **Java的线程共享机制，最重要的是Thread中的ThreadLocalMap，ThreadLocal其实不重要，它只是一个钩子，**尝试从Thread中钩出ThreadLocalMap，东西实际被存在每一个Thread的ThreadLocalMap中，所以广义上可以理解为东西是存在Thread中

- **ThreadLocal其实不存东西，ThreadLocalMap的key也不是Thread**

# 引用类型

- Java中共有4种引用类型，与Garbage Collection相关
  - 强引用：强不受GC影响，除非引用全部切断：Student s = new Student()，假设当前只有s指向Student对象，当s=null时，Student对象会在下次GC时被回收
  - 软引用：会在内存不足触发GC时被回收（适用于高速缓存）
  - **弱引用：每次GC时都回收，不论内存是否不足**
  - 虚引用：堆外内存，比如zerocopy
- ThreadLocalMap的Entry继承的弱引用，但其实key才是弱引用，Entry本身不是
- 这么做的理由：外部强引用ThreadLocal，如果Map中还是强引用，那这个ThreadLocal仍然无法彻底释放；改为弱引用后只要外部强引用全部切断，下次GC时Map中的key指向的ThreadLocal就会被回收，防止ThreadLocal内存泄露
- 但是这样又会导致Entry内存泄露，因为Entry的key为null了，为了解决这个我呢提，ThreadLocalMap在每次get/set之前都会判断key是否为null，如果为null则将value也设为null
- 但上述做法只能在第二次访问时才生效，如果一直不访问则一直都不会清除，最保险的做法时每次使用完毕都自己清理