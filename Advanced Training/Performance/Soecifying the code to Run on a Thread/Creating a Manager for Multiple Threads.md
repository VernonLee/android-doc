Creating a Manager for Multiple Threads 为多线程创建管理者

之前章节讲述如何定义在分离的线程中执行的任务。如果任务仅被执行一次，那么看完这章就懂了。如果使用单个任务重复运行不同数据集，但是在某个时刻只能有一个任务被运行，IntentService适合需要。在资源可用的时候自动运行任务，或者允许多个线程同时运行，需要提供线程集合的管理。为了达到这个目的，使用ThreadPoolExecutor实例，当它内部线程池中某个线程空闲的时候，从队列中调度任务并运行。为了执行任务，所做的是必须将它添加队列中。

线程池可以并行运行任务，所以确保代码线程安全。在synchronized块内，封闭的变量可以被多个线程线程访问。这种实现目的在于当一个线程修改变量的时候阻止另外一个变量访问它。典型的，这种情况产生于静态变量，但是也发生在任何只被初始化一次的对象。更多详细信息 阅读 Processes and Threads API 指导。

Define the Thread Pool Class 定义线程池类

在类内部初始化ThreadPoolExecutor. 在类内部，按照下面做法：

为线程池使用静态变量

应用中线程池实例只有一个。为了有限的CPU或网络资源的使用单个控制权。如果有不同Runnable类型，为每种类型分配一个线程池，但是每个线程池也只有单个实例。比如，添加以下部分作为全局变量的声明：

public class PhotoManager {
	...
	static {
	 ...
	 // 创建PhotoManager单个实例
	 sInstance = new PhotoManager();
	}
...

使用私有构造器

设置构造器私有，保证类是单例，意味不需要在synchronized块内封闭类的访问：
public class PhotoManager {
	/**
	* 构造工作队列和线程池用于下载和解码图片。因为构造器是私有的，
	* 对其他类不可用，即使在同一个包内。
	*/
	private PhotoManager() {
	 ...
	}

在线程池内部调用方法启动任务

在线程池类内部定义添加任务到线程池队列的方法。比如：
public class PhotoManager {
	...
	// 由PhotoView调用获取图片
	static public PhotoTask startDownload(
		PhotoView imageView,
		boolean cacheFlag) {
		...
		// 添加任务到线程池并运行
		sInstance.mDownloadThreadPool.execute(downloadTask.
			getHTTPDownloadRunnable());
		...
		}

在构造器内部初始化Handler并绑定到应用UI线程

Handler允许应用安全调用UI对象，比如View对象, 的方法.大多数UI对象只能在UI线程中安全修改。在Communicate with the UI Thread中有详细描述。比如：
private PhotoManager() {
	...
	// 定义Handler对象并附加给UI线程
	mHandler = new Handler(Looper.getMainLooper()) {
		// handleMessage()中定义当Handler收到新消息的时候要执行的操作
		public void handleMessage(Message msg) {
		 ...
		}
	..
	}
}

Determine the Thread Pool Parameters 确定线程池参数

一旦有了综合的类结构，开始定义线程池。为了初始化ThreadPoolExecutor对象，需要下面值：

初始化线程池尺寸和最大值

分配给线程池中线程的数量。并且最多允许的数量。线程池中线程的数量根据设备可用内核数量而定。下面是获取系统环境中可用内核数量：
public class PhotoManager {
	...
	// 获取可用内核数量(有时和和最大可用值不一样)
	private static int NUMBER_OF_CORES =
		Runtime.getRuntime().availableProcessors();
}

该数目可能不是设备物理内核的数量；一些设备根据系统负载恢复一个或多个CPU内核。对于这些设备，availableProcessors()返回活跃内核数目，可能少于系统内核总数。


Keep alive and time unit 保持在线和时间单元

线程将保持空闲直到被停掉的持续时间。该持续时间由时间单元值决定，该值是在TimeUnit中定义的常量之一。

A queue of tasks 队列任务

即将到达的队列从ThreadPoolExecutor中选择Runnable对象。为了启动线程，线程池管理者从先进先出队列中拿到Runnable对象并且添加到线程中。当创建线程池的时候提供队列对象，使用任何实现BlockingQueue接口的队列类。为了匹配应用要求，可以从可用队列实现中选择；为了了解更对关于他们欣喜，查看ThreadPoolExecutor类。下面是使用LinkedBlockQueue类例子：

public class PhotoManager {
	...
	private PhotoManager() {
		...
		// A queue of Runnables
		private final BlockingQueue<Runnable> mDecodeWorkQueue;
		...
		// Instance the queue of Runnables as a LinkedBlockingQueue
		mDecodeWorkQueue = new LindedBlockingQueue<Runnable>();
		...
	}
	...
}

Create a Pool of Threads 创建线程池

为了创建线程池，调用ThreadPoolExecutor()方法初始化线程池管理者。它创建和管理一个有限的线程组。因为初始的尺寸和最大尺寸都相同，ThreadPoolExecutor在被初始化后创建所有线程对象。比如：
private PhotoManager() {
	...
	// set the amount of time an idle thread waits before terminating
	private static final int KEEP_ALIVE_TIME = 1;
	// set the time unit to seconds
	private static final TimeUnit KEEP_ALIVE_UNIT = TimeUnit.SECOND;
	// create a thread pool manager
	mDecodeThreadPool = new ThreadPoolExecutor(
		NUMBER_OF_CORES, // 初始的线程池尺寸
		NUMBER_OF_CORES, // 最大线程池尺寸
		KEEP_ALIVE_TIME,
		KEEP_ALIVE_TIME_UNIT,
		mDecodeWorkQueue)； 
}



