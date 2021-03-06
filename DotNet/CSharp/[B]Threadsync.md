# 基元线程同步
**线程同步带来的问题**
- 比较繁锁
- 影响性能
- 一次只允许一个线程访问资源

**FCL类库特点**
所有静态方法都线程安全，所有实例方法都非线程安全
**基元**：指在代码中使用的最简单的构造。
注：基元用户模式要显著已于内核模式。
   模式切换严重影响性能，但会避免其浪费CPU时间。
**CLR原子性读取的变量类型有**：Boolean,Char,(S)Byte,(U)Int16,(U)Int32,(U)IntPtr,Single及引用类型。
注：long有可能非原子性读取或写入
**易变构造（Volatile）及互锁构造**
原子性读取或简单类型
注：调试器会进行优化，从来出现一些调试版本不会出现但发行版本才会出现的问题。
**Volatile.Write及Volatile.Read**
按照编码顺序，之前的加载和存储必须在调用Volatile.Write之前发生；
按照编码顺序，之后的加载和存储必须在调用Volatile.Read之后发生。
Volatile关键字告诉C#和JIT编译器不将字段缓存到CPU的寄存器中，确保字段的所有读写都在RAM中进行。
**简单的自旋锁**
**SpinLock**
获取不到资源时，循环等待获取资源。等待时使用SpinWait，强制切换到另外一个线程。

**使用InterLocked.CompareExchange实现原子Maximun操作**
```cs
        public static Int32 Maximum(ref Int32 target, Int32 value)
        {
            Int32 currentVal = target, startVal, desiredVal;
            do
            {
                startVal = currentVal;
                desiredVal = Math.Max(startVal, value);
                currentVal = Interlocked.CompareExchange(ref target, desiredVal, startVal);
            } while (startVal != currentVal);
            return desiredVal;
        }
```
### 内核模式构造
比用户模式慢的多的一个原因就是它要求操作系统的配合，及对内核对象上调用的每个方法都造成托管代码到本机用户代码的黑气，现反转换，这需要大量的CPU时间。内核构造用**事件**、**信号量**、**互斥体**。
#### WaitHandle
包装WIndows内核对象句柄，类层次结构如下：
WaitHandle
- EventWaitHandle
  - AutoResetEvent
  - ManualResetEvent
- Semaphore
- Mutex

# 混合线程同步
## Monitor
支持自旋、线程所有权、递归的互斥锁。
锁定时需要注意的地方：
1. 不要在方法中锁定对象本身，否则可能在别人在外锁定该对象时出现死锁现象。坚持使用私有锁。
2. 锁定派生自MarshalByRefObject的对象时需要特别注意，锁定的可能只是代理对象。
3. 不要锁定类型对象本身，因为类型对象为多个应用程序域共享，使用到的地方都会进行同步。
4. 不要锁定字符串，字符串留用会使两段独立的代码以同步方式执行。
5. 不要锁定值类型，值类型的装箱会使锁定无效。
6. [MethodImpl(MethodImplOptions.Synchronized)]特定会在实例方法中添加的锁定对象本身，会在静态方法中锁定类型对象。
7. 类型构造器调用时，会获取类型对象上的一个锁，该类型可能以‘AppDomain中立’的方式加载，所以会出问题。

### ReaderWriterLockSlim
其有一个参数LockRecurionsPolicy，建议传递LockRecursionPolicy.NoRecursion。
- 读写互斥
- 写写互斥

**ReaderWriterLock**
这个构造有多种问题，即使没有竞争，速度也很慢。线程所有权及递归行为是这个构造加强的，完全无法取消，这使锁变的更慢。
### OneManyLock
CLR Via C#作者编写的一个reader-writer构造（Page 709），它的速度比FCL的ReaderWriterLockSlim快。  
### CountdownEvent
内部使用ManualResetEventSlim对象，这个构造阻塞一个线程，直到它的内部计数器变为0，某种角度上讲，这个构造行为和Semaphore的行为相反。
注：一量CountdownEvent的CurrentCount变成0，它就不能更改了。
### Barrier
使用场景比较少，主要用于多个任务不同阶段。只有多个任务的同一阶段完成才能进入下一个阶段。示例如下：
```cs
class Program
    {
        private static void Phase0Doing(int TaskID)
        {
            Console.WriteLine("Task : #{0}   =====  Phase 0", TaskID);
        }

        private static void Phase1Doing(int TaskID)
        {
            Console.WriteLine("Task : #{0}   *****  Phase 1", TaskID);
        }

        private static void Phase2Doing(int TaskID)
        {
            Console.WriteLine("Task : #{0}   ^^^^^  Phase 2", TaskID);
        }

        private static void Phase3Doing(int TaskID)
        {
            Console.WriteLine("Task : #{0}   $$$$$  Phase 3", TaskID);
        }

        private static int _TaskNum = 4;
        private static Task[] _Tasks;
        private static Barrier _Barrier;


        static void Main(string[] args)
        {
            _Tasks = new Task[_TaskNum];
            _Barrier = new Barrier(_TaskNum, (barrier) =>
            {
                Console.WriteLine("-------------------------- Current Phase:{0} --------------------------", 
                                  _Barrier.CurrentPhaseNumber);
            });

            for (int i = 0; i < _TaskNum; i++)
            {
                _Tasks[i] = Task.Factory.StartNew((num) =>
                {
                    var taskid = (int)num;

                    Phase0Doing(taskid);
                    _Barrier.SignalAndWait();

                    Phase1Doing(taskid);
                    _Barrier.SignalAndWait();

                    Phase2Doing(taskid);
                    _Barrier.SignalAndWait();

                    Phase3Doing(taskid);
                    _Barrier.SignalAndWait();

                }, i);
            }

            var finalTask = Task.Factory.ContinueWhenAll(_Tasks, (tasks) =>
                {
                    Task.WaitAll(_Tasks);
                    Console.WriteLine("========================================");
                    Console.WriteLine("All Phase is completed");

                    _Barrier.Dispose();
                });

            finalTask.Wait();

            Console.ReadLine();
        }
    }
```
###  同步构造小节
将数据从一个线程交给另一个线程时，避免多个线程同时访问数据，不能做到时，尽量使用Volatile和Interlocked方法，速度很快，绝不阻塞线程。
### Singleton
```cs
internal static class Singletons {
   public static class V1 {
      public sealed class Singleton {
         // s_lock is required for thread safety and having this object assumes that creating  
         // the singleton object is more expensive than creating a System.Object object and that 
         // creating the singleton object may not be necessary at all. Otherwise, it is more  
         // efficient and easier to just create the singleton object in a class constructor
         private static readonly Object s_lock = new Object();

         // This field will refer to the one Singleton object
         private static Singleton s_value = null;

         // Private constructor prevents any code outside this class from creating an instance 
         private Singleton() { /* ... */ }

         // Public, static method that returns the Singleton object (creating it if necessary) 
         public static Singleton GetSingleton() {
            // If the Singleton was already created, just return it (this is fast)
            if (s_value != null) return s_value;

            Monitor.Enter(s_lock);  // Not created, let 1 thread create it
            if (s_value == null) {
               // Still not created, create it
               Singleton temp = new Singleton();

               // Save the reference in s_value (see discussion for details)
               Volatile.Write(ref s_value, temp);
            }
            Monitor.Exit(s_lock);

            // Return a reference to the one Singleton object 
            return s_value;
         }
      }
   }

   public static class V2 {
      public sealed class Singleton {
         private static Singleton s_value = new Singleton();

         // Private constructor prevents any code outside this class from creating an instance 
         private Singleton() { }

         // Public, static method that returns the Singleton object (creating it if necessary) 
         public static Singleton GetSingleton() { return s_value; }
      }
   }

   public static class V3 {
      public sealed class Singleton {
         private static Singleton s_value = null;

         // Private constructor prevents any code outside this class from creating an instance 
         private Singleton() { }

         // Public, static method that returns the Singleton object (creating it if necessary) 
         public static Singleton GetSingleton() {
            if (s_value != null) return s_value;

            // Create a new Singleton and root it if another thread didn’t do it first
            Singleton temp = new Singleton();
            Interlocked.CompareExchange(ref s_value, temp, null);

            // If this thread lost, then the second Singleton object gets GC’d

            return s_value; // Return reference to the single object
         }
      }
   }
```
注：V1版本中`Volatile.Write(ref s_value, temp)`，而没有这样写`s_value = new Singleton()`（大多数人会这样写），我们认为编译器会为Singleton分配空间，调用构造器，再将引用赋值给s_value,==但实际上，编译器，并不是这样的执行顺序，先是分配内存，再将引用发布到s_value，再调用构造器，假设这时候有其它线程使用该对象，可以的情况是其构造器还未调用，从而出错bug，而该问题又是设计上的错误，很难追踪==。
**Lazy<T> and LazyInitializer**
使用这两个关键字也可以实现单例，若不希望构建Lazy包装对象，可使用LazyInitializer。
```cs
 public static class V4 {
     public sealed class Singleton
    {
        private static Singleton _instance = null;

        private Singleton() { }

        public static Singleton GetSingleton()
        {
            return LazyInitializer.EnsureInitialized(ref _instance, () => new Singleton());
        }
    }
   }```
### 条件变量模式
```cs
internal static class ConditionVariables {
   public sealed class ConditionVariablePattern {
      private readonly Object m_lock = new Object();
      private Boolean m_condition = false;

      public void Thread1() {
         Monitor.Enter(m_lock);        // Acquire a mutual-exclusive lock

         // While under the lock, test the complex condition "atomically"
         while (!m_condition) {
            // If condition is not met, wait for another thread to change the condition
            Monitor.Wait(m_lock);	   // Temporarily release lock so other threads can get it
         }

         // The condition was met, process the data...

         Monitor.Exit(m_lock);         // Permanently release lock
      }

      public void Thread2() {
         Monitor.Enter(m_lock);        // Acquire a mutual-exclusive lock

         // Process data and modify the condition...
         m_condition = true;

         // Monitor.Pulse(m_lock);	   // Wakes one waiter AFTER lock is released
         Monitor.PulseAll(m_lock);	   // Wakes all waiters AFTER lock is released

         Monitor.Exit(m_lock);         // Release lock
      }
   }
```
```cs
   internal static class ConditionVariables {
   public sealed class ConditionVariablePattern {
      private readonly Object m_lock = new Object();
      private Boolean m_condition = false;

      public void Thread1() {
         Monitor.Enter(m_lock);        // Acquire a mutual-exclusive lock

         // While under the lock, test the complex condition "atomically"
         while (!m_condition) {
            // If condition is not met, wait for another thread to change the condition
            Monitor.Wait(m_lock);	   // Temporarily release lock so other threads can get it
         }

         // The condition was met, process the data...

         Monitor.Exit(m_lock);         // Permanently release lock
      }

      public void Thread2() {
         Monitor.Enter(m_lock);        // Acquire a mutual-exclusive lock

         // Process data and modify the condition...
         m_condition = true;

         // Monitor.Pulse(m_lock);	   // Wakes one waiter AFTER lock is released
         Monitor.PulseAll(m_lock);	   // Wakes all waiters AFTER lock is released

         Monitor.Exit(m_lock);         // Release lock
      }
   }
```
### Task优点
- 使用内存比线程少，创建和销毁时间少；
- 线程池根据可用CPU数量自动伸缩任务规模；
- 循环利用
- 线程池可以更好的调度，减少进程中的线程数，减少上下文切换

### 异步线程同步
锁用的地方很多，但长时间拥有锁会带来巨大的伸缩性问题，如果得不到锁，直接返回而不应阻塞，当锁可用时，再恢复执行即可。
**SemaphoreSlim**通过**WaitAsync**实现了这个思路，独占访问资源，不会阻塞线程。如下代码：
```cs
   /// <summary>
   /// Indicates if the OneManyLock should be acquired for exclusive or shared access.
   /// </summary>
   public enum OneManyMode {
      /// <summary>
      /// Indicates that exclusive access is required.
      /// </summary>
      Exclusive,

      /// <summary>
      /// Indicates that shared access is required.
      /// </summary>
      Shared
   }
internal static class AsyncSynchronization {
   public static void Go() {
      SemaphoreSlimDemo();
   }

   private static void SemaphoreSlimDemo() {
      SemaphoreSlim asyncLock = new SemaphoreSlim(1, 1);
      List<Task> tasks = new List<Task>();
      for (Int32 op = 0; op < 5; op++) {
         var capturedOp = op;
         tasks.Add(Task.Run(() => AccessResourceViaAsyncSynchronization(asyncLock, capturedOp)));
         Thread.Sleep(200);
      }
      Task.WaitAll(tasks.ToArray());
      Console.WriteLine("All operations done");
      Console.ReadLine();
   }

   private static async Task AccessResourceViaAsyncSynchronization(SemaphoreSlim asyncLock, Int32 operation) {
      // Execute whatever code you want here...
      Console.WriteLine("ThreadID={0}, OpID={1}, await for {2} access",
         Environment.CurrentManagedThreadId, operation, "exclusive");
      await asyncLock.WaitAsync();     // Request exclusive access to a resource via its lock
      // When we get here, we know that no other thread his accessing the resource
      // Access the resource (exclusively)...
      Console.WriteLine("ThreadID={0}, OpID={1}, got access at {2}",
         Environment.CurrentManagedThreadId, operation, DateTime.Now.ToLongTimeString());
      Thread.Sleep(5000);

      // When done accessing resource, relinquish lock so other code can access the resource
      asyncLock.Release();

      // Execute whatever code you want here...
   }
```
.NET Framework还提供了**ConcurrentExclusiveSchedulerPair**类，内部有两个任务调度器，一个负责Writer语义，一个负责Reader语义。只要没有ExclusiveScheduler调度的任务，ConcurrentScheduler的任务就可以同时进行。
```cs
   private static void ConcurrentExclusiveSchedulerDemo() {
      var cesp = new ConcurrentExclusiveSchedulerPair();
      var tfExclusive = new TaskFactory(cesp.ExclusiveScheduler);
      var tfConcurrent = new TaskFactory(cesp.ConcurrentScheduler);

      List<Task> tasks = new List<Task>();
      for (Int32 operation = 0; operation < 5; operation++) {
         var capturedOp = operation;
         var exclusive = operation < 2; 
         Task t = (exclusive ? tfExclusive : tfConcurrent).StartNew(() => {
            Console.WriteLine("ThreadID={0}, OpID={1}, {2} access",
               Environment.CurrentManagedThreadId, capturedOp, exclusive ? "exclusive" : "concurrent");            
            Thread.Sleep(5000);
         });

         tasks.Add(t);
         Thread.Sleep(200);
      }
      Task.WaitAll(tasks.ToArray());
      Console.WriteLine("All operations done");
      Console.ReadLine();
}
```
但该类在创建多个任务后仍然会只是对多个线程进行了同步，阻塞了线程，CLR via C#的作者提供了这样一个类来解决这个问题AsyncOneManyLock类，代码如下：
```cs
/// <summary>
   /// This class implements a reader/writer lock that never blocks any threads.
   /// To use, await the result of AccessAsync and, after manipulating shared state,
   /// call Release.
   /// </summary>
   public sealed class AsyncOneManyLock {
      #region Lock code
      private SpinLock m_lock = new SpinLock(true);   // Don't use readonly with a SpinLock
      private void Lock() { Boolean taken = false; m_lock.Enter(ref taken); }
      private void Unlock() { m_lock.Exit(); }
      #endregion

      #region Lock state and helper methods
      private Int32 m_state = 0;
      private Boolean IsFree { get { return m_state == 0; } }
      private Boolean IsOwnedByWriter { get { return m_state == -1; } }
      private Boolean IsOwnedByReaders { get { return m_state > 0; } }
      private Int32 AddReaders(Int32 count) { return m_state += count; }
      private Int32 SubtractReader() { return --m_state; }
      private void MakeWriter() { m_state = -1; }
      private void MakeFree() { m_state = 0; }
      #endregion

      // For the no-contention case to improve performance and reduce memory consumption
      private readonly Task m_noContentionAccessGranter;

      // Each waiting writers wakes up via their own TaskCompletionSource queued here
      private readonly Queue<TaskCompletionSource<Object>> m_qWaitingWriters =
         new Queue<TaskCompletionSource<Object>>();

      // All waiting readers wake up by signaling a single TaskCompletionSource
      private TaskCompletionSource<Object> m_waitingReadersSignal =
         new TaskCompletionSource<Object>();
      private Int32 m_numWaitingReaders = 0;

      /// <summary>Constructs an AsyncOneManyLock object.</summary>
      public AsyncOneManyLock() {
         m_noContentionAccessGranter = Task.FromResult<Object>(null);
      }

      /// <summary>
      /// Asynchronously requests access to the state protected by this AsyncOneManyLock.
      /// </summary>
      /// <param name="mode">Specifies whether you want exclusive (write) access or shared (read) access.</param>
      /// <returns>A Task to await.</returns>
      public Task WaitAsync(OneManyMode mode) {
         Task accressGranter = m_noContentionAccessGranter; // Assume no contention

         Lock();
         switch (mode) {
            case OneManyMode.Exclusive:
               if (IsFree) {
                  MakeWriter();  // No contention
               } else {
                  // Contention: Queue new writer task & return it so writer waits
                  var tcs = new TaskCompletionSource<Object>();
                  m_qWaitingWriters.Enqueue(tcs);
                  accressGranter = tcs.Task;
               }
               break;

            case OneManyMode.Shared:
               if (IsFree || (IsOwnedByReaders && m_qWaitingWriters.Count == 0)) {
                  AddReaders(1); // No contention
               } else { // Contention
                  // Contention: Increment waiting readers & return reader task so reader waits
                  m_numWaitingReaders++;
                  accressGranter = m_waitingReadersSignal.Task.ContinueWith(t => t.Result);
               }
               break;
         }
         Unlock();

         return accressGranter;
      }

      /// <summary>
      /// Releases the AsyncOneManyLock allowing other code to acquire it
      /// </summary>
      public void Release() {
         TaskCompletionSource<Object> accessGranter = null;   // Assume no code is released

         Lock();
         if (IsOwnedByWriter) MakeFree(); // The writer left
         else SubtractReader();           // A reader left

         if (IsFree) {
            // If free, wake 1 waiting writer or all waiting readers
            if (m_qWaitingWriters.Count > 0) {
               MakeWriter();
               accessGranter = m_qWaitingWriters.Dequeue();
            } else if (m_numWaitingReaders > 0) {
               AddReaders(m_numWaitingReaders);
               m_numWaitingReaders = 0;
               accessGranter = m_waitingReadersSignal;

               // Create a new TCS for future readers that need to wait
               m_waitingReadersSignal = new TaskCompletionSource<Object>();
            }
         }
         Unlock();

         // Wake the writer/reader outside the lock to reduce
         // chance of contention improving performance
         if (accessGranter != null) accessGranter.SetResult(null);
      }
   }
}
```
**该类的使用实例**
```cs
   private static void OneManyDemo() {
      var asyncLock = new AsyncOneManyLock();
      List<Task> tasks = new List<Task>();
      for (Int32 x = 0; x < 5; x++) {
         var y = x;

         tasks.Add(Task.Run(async () => {
            var mode = (y < 3) ? OneManyMode.Shared : OneManyMode.Exclusive;
            Console.WriteLine("ThreadID={0}, OpID={1}, await for {2} access",
               Environment.CurrentManagedThreadId, y, mode);
            var t = asyncLock.WaitAsync(mode);
            await t;
            Console.WriteLine("ThreadID={0}, OpID={1}, got access at {2}",
               Environment.CurrentManagedThreadId, y, DateTime.Now.ToLongTimeString());
            Thread.Sleep(5000);
            asyncLock.Release();
         }));
         Thread.Sleep(200);
      }
      Task.WaitAll(tasks.ToArray());
      Console.WriteLine("All operations done");
      Console.ReadLine();
   }
```
### 并发集合类
1. ConcurrentQueue
2. ConcurrentStack
3. ConcurrentBag //无序数据集合，可以重复
4. ConcurrentDictionary  

前三个并发集合都实现了IProducerConsumerCOllection，可以将这三个非阻塞集合变成阻塞的生产者消费者集合
```cs
internal static class BlockingCollectionDemo {
   public static void Go() {
      var bl = new BlockingCollection<Int32>(new ConcurrentQueue<Int32>());

      // A thread pool thread will do the consuming
      ThreadPool.QueueUserWorkItem(ConsumeItems, bl);

      // Add 5 items to the collection
      for (Int32 item = 0; item < 5; item++) {
         Console.WriteLine("Producing: " + item);
         bl.Add(item);
      }

      // Tell the consuming thread(s) that no more items will be added to the collection
      bl.CompleteAdding();

      Console.ReadLine();  // For testing purposes
   }

   private static void ConsumeItems(Object o) {
      var bl = (BlockingCollection<Int32>)o;

      // Block until an item shows up, then process it
      foreach (var item in bl.GetConsumingEnumerable()) {
         Console.WriteLine("Consuming: " + item);
      }

      // The collection is empty and no more items are going into it
      Console.WriteLine("All items have been consumed");
   }
}
```
