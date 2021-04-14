#### Happens-Before规则

**定义**：前面一个操作的结果对后续操作是可见的

规则如下

1. 程序的顺序性

   ```java
   // 以下代码
   class VolatileExample {
           int x = 0;
           volatile boolean v = false;
           public void writer() {
             x = 42;
             v = true;
           public void reader() {
             if (v == true) {
   // 这里 x 会是多少呢? }
   } }
   ```

   这个规则值得是一个线程中，按照程序顺序，前面的操作happens-Before于后续的任意操作。第6行“x=42”，happens-before于第七行代码。程序前面对某个变量的修改一定是对后续操作可见的。

2. volatile变量规则

   这条规则是指对一个volatile变量的写操作，happens-before于后续对这个volatile变量的读。结合规则3一起考虑

3. 传递性

   这条规则是指A happens-before B,且B happens-before C，那么A Happens-before C。将这条规则应用于规则一的代码中

   - “x=42” happens-before 写变量 “v=true”
   - 写变量 v=true happens-before 读变量 ”v== true“

   如果线程 B 读到了“v=true”，那么线程 A 设置的“x=42”对线程 B 是可见的。也就是 说，线程 B 能看到 “x == 42”

4. 管程中锁的规则

   这条规则是指指的是前一个线程的解锁操作对后一个线程的加锁操作可见

   要理解这个规则，就首先要了解“管程指的是什么”。**管程**是一种通用的同步原语，在 Java 中指的就是 synchronized，synchronized 是 Java 里对管程的实现。管程中的锁在 Java 里是隐式实现的，例如下面的代码，在进入同步块之前，会自动加锁， 而在代码块执行完会自动释放锁，加锁以及释放锁都是编译器帮我们实现的。

   ```java
   synchronized(this){
     // x为共享变量 初始值 10
     if(this.x < 12){
       this.x = 12
     }
   }
   ```

   假设 x 的初始值是 10，线程 A 执行 完代码块后 x 的值会变成 12(执行完自动释放锁)，线程 B 进入代码块时，能够看到线程 A 对 x 的写操作，也就是线程 B 能够看到 x==12。这个也是符合我们直觉的，应该不难理 解。
   
5. 线程start（）规则

   这条是关于线程启动的。它是指主线程 A 启动子线程 B 后，子线程 B 能够看到主线程在启动子线程 B 前的操作。
   
   换句话说就是，如果线程 A 调用线程 B 的 start() 方法(即在线程 A 中启动线程 B)，那 么该 start() 操作 Happens-Before 于线程 B 中的任意操作。具体可参考下面示例代码。
   
   ```java
   Thread B = new Thread(()->{
   // 主线程调用 B.start() 之前
   // 所有对共享变量的修改，此处皆可见 // 此例中，var==77
   });
   // 此处对共享变量 var 修改 var = 77;
   // 主线程启动子线程 B.start();
   ```
   
6. 线程join（）规则
   
   这条是关于线程等待的。它是指主线程 A 等待子线程 B 完成(主线程 A 通过调用子线程 B 的 join() 方法实现)，当子线程 B 完成后(主线程 A 中 join() 方法返回)，主线程能够看 到子线程的操作。当然所谓的“看到”，指的是对**共享变量**的操作。
   
   换句话说就是，如果在线程 A 中，调用线程 B 的 join() 并成功返回，那么线程 B 中的任意 操作 Happens-Before 于该 join() 操作的返回。具体可参考下面示例代码。
   
   ```java
    Thread B = new Thread(()->{
   2 // 此处对共享变量 var 修改
   3 var = 66;
   4 });
   5 // 例如此处对共享变量修改，
   6 // 则这个修改结果对线程 B 可见
   7 // 主线程启动子线程
   8 B.start();
   9 B.join()
   10 // 子线程所有对共享变量的修改
   11 // 在主线程调用 B.join() 之后皆可见
   12 // 此例中，var==66
   ```
   
   
   
   