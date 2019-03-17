https://mp.csdn.net/mdeditor/88622219#

Netty 是如何工作的？
因为代码中没有异步的代码。那非阻塞是如何实现的？

# 一个 NIO 是不是只能有一个 selector ？
可以有多个 selector
# selector 是不是只能注册一个 ServerSocketChannel？
可以注册多个
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190317104809704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NhcnJ5TmV0,size_16,color_FFFFFF,t_70)

- 目录结构
![目录结构](https://img-blog.csdnimg.cn/20190317141759804.png)
## 初始化服务
### Start 
```java
//初始化线程
		NioSelectorRunnablePool nioSelectorRunnablePool = new NioSelectorRunnablePool(Executors.newCachedThreadPool(), Executors.newCachedThreadPool());
		
		//获取服务类 
		ServerBootstrap bootstrap = new ServerBootstrap(nioSelectorRunnablePool);
		
		//绑定端口
		//初始化 Channel
		//线程池 获取 boss select
		//selverChanner 注册到 selector , key 为 accept 状态
		bootstrap.bind(new InetSocketAddress(10101));
		
		System.out.println("start");
```
1. init 线程池
boss 线程池 
woker 线程池
每个线程一个 selector

2. selector implements Runnable
1个selector 有 1个Queue<Runnable> taskQueue
 每个 selector 都公用 boss pool or worker pool
 
3. 每一个 selector 获取 Selector  并启动线程  ,在 AbstractNioSelector 中执行 run 方法。
```java
/**
	 * 获取selector并启动线程
	 */
	private void openSelector() {
		try {
			this.selector = Selector.open();
		} catch (IOException e) {
			throw new RuntimeException("Failed to create a selector.");
		}
		executor.execute(this);
	}
	run :
while (true) {
			try {
				wakenUp.set(false);

				select(selector);

				processTaskQueue();

				process(selector);
			} catch (Exception e) {
				// ignore
			}
		}
```





-------
### Boss or Worker 注册不同 Selector 的特有接口

```java
public interface Boss {
	
	/**
	 * 加入一个新的ServerSocket
	 * @param serverChannel
	 */
	public void registerAcceptChannelTask(ServerSocketChannel serverChannel);
}
```

### NioSelectorRunnablePool
```java
/**
	 * boss线程数组
	 */
	private final AtomicInteger bossIndex = new AtomicInteger();
	private Boss[] bosses;

	/**
	 * worker线程数组
	 */
	private final AtomicInteger workerIndex = new AtomicInteger();
	private Worker[] workeres;
```

  
1. 初始化线程池
  2. init boss  创建NIOServerBoss
```java
private void initBoss(Executor boss, int count) {
		this.bosses = new NioServerBoss[count];
		for (int i = 0; i < bosses.length; i++) {
			bosses[i] = new NioServerBoss(boss, "boss thread " + (i+1), this);
		}
	}
```

  3. initWorker  创建
```java
private void initWorker(Executor worker, int count) {
		this.workeres = new NioServerWorker[count];
		for (int i = 0; i < workeres.length; i++) {
			workeres[i] = new NioServerWorker(worker, "worker thread " + (i+1), this);
		}
	}
```
 
  4.  注册并激活selector 
  当 boss 加入一个新的SeverSocket 时 //负责监听端口 channel
  当 worker channel 加入 selector 时 // 负责处理具体的 事件 channel
  
```java
/**
	 * 注册一个任务并激活selector
	 * 
	 * @param task
	 */
	protected final void registerTask(Runnable task) {
		taskQueue.add(task);

		Selector selector = this.selector;

		if (selector != null) {
			//if wakeUp 是false那么 update为 true
			//False return indicates that the actual value was not equal to the expected value.
			if (wakenUp.compareAndSet(false, true)) {
				selector.wakeup();//唤醒阻塞的 selector
			}
		} else {
			taskQueue.remove(task);
		}
	}
```

  5.

 
### ServerBootstrap
服务类，绑定端口
### NioServerBoss
NioServerBoss extends AbstractNioSelector implements Boss

### NioServerWorker
extends AbstractNioSelector implements Worker

### AbstractNioSelector  implements Runnable 最顶层的selector
- 初始化 
- 每个线程都有 selector 的能力
```java
	AbstractNioSelector(Executor executor, String threadName, NioSelectorRunnablePool selectorRunnablePool) {
		this.executor = executor;
		this.threadName = threadName;
		this.selectorRunnablePool = selectorRunnablePool;
		openSelector();
	}
	
	/**
	 * 获取selector并启动线程
	 */
	private void openSelector()  {
		try {
			this.selector = Selector.open(); 
		} catch (IOException e) {
			throw new RuntimeException("Failed to create a selector.");
		}
		executor.execute(this);//执行当前线程对象，运行下面的 run方法
	}
```
- 核心！！！！！！！
```java
	@Override
	public void run() {
		
		Thread.currentThread().setName(this.threadName);

		while (true) {
			try {
			    //设置select 阻塞状态，当 registerTask 方法执行时醒来，处理。
				wakenUp.set(false); 
				 
				//抽象方法
				//boss 选择阻塞 自定义
				//worker 选择阻塞500ms 自定义
				select(selector); 
				
				//执行任务队列中 Runnable 对象的run ，注册 channel到 boss or worker 等待处理的对象。
				processTaskQueue();
                
                //处理 selector
                //boss 非阻塞接收事件，然后 register 到 worker 的队列。
                //worker 具体处理事件，与返回事件
				process(selector);
			} catch (Exception e) {
				// ignore
			}
		}
	}
```
| boss  |  worker  |
|:-----|---|
| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190317150244668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NhcnJ5TmV0,size_16,color_FFFFFF,t_70)|![在这里插入图片描述](https://img-blog.csdnimg.cn/20190317150444768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NhcnJ5TmV0,size_16,color_FFFFFF,t_70) |

- 注册一个任务并激活selector
```java
/**
	 * 注册一个任务并激活selector
	 * 
	 * @param task
	 */
	protected final void registerTask(Runnable task) {
		taskQueue.add(task);

		Selector selector = this.selector;

		if (selector != null) {
			//if wakeUp 是false那么 update为 true
			//False return indicates that the actual value was not equal to the expected value.
			if (wakenUp.compareAndSet(false, true)) {
				selector.wakeup();//唤醒阻塞的 selector
			}
		} else {
			taskQueue.remove(task);
		}
	}
```

# 存在疑问？
NIO 如何提高的效率？
如何多路复用？
见下次。