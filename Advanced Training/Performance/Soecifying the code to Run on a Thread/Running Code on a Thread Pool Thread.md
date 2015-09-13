Running Code on a Thread Pool Thread 在线程池中运行代码

之前章节讲述如何定义类管理线程池和任务并且启动他们。这节课讲述如何在线程池中运行任务。为了实现这个，添加任务到线程池的工作队列。当某个线程可用，ThreadPoolExecutor从队列中取出任务在线程中运行。

这节课中讲述如何终止正在运行的任务。当启动某个任务后，发现它并没有什么作用，所以需要停止该任务。而不是浪费处理器时间，可以在任务正在运行的时候取消线程。比如说，如果从网络中下载图片并且使用缓存，检测出某个图片已经在缓存中存在。根据如何编写应用，在启动下载之前或许不能检测。


Run a Task on a Thread in the Thread Pool 在线程池中运行任务

启动线程池中任务对象，传递Runnable给ThreadPoolExecutor.execute(). 该调用添加任务到线程池的工作队列。当空闲线程可用，线程管理者调度等待时间最长的那个在线程中运行：

public class PhotoManager {
	public void handleState(PhotoTaks photoTask, int state) {
		switch(state) {
		// the task finished dowloading the image
		case DOWNLOAD_COMPLTED:
		// decodes the image
		mDecodeThreadPool.execute(
			photoTask.getPhotoDecodeRunnable());
		}
	}
}

当ThreadPoolExecutor在线程中启动Runnable, 自定调用Runnable对象的run()方法。


Interrupt Running Code 中断运行

为了终止任务，需要中断任务的线程。准备操作之前，当创建任务的时候需要存储任务线程的句柄。比如：

class PhotoDecodeRunnable implements Runnable {
	// defines the code to run for this task
	public void run() {
		// 在包含PhotoDecodeRunnable的对象中存储当前线程
		mPhotoTask.setImageDecodeThread(Thread.currentThread());
	}
}

为了终止线程，调用Thread.interrupt()方法。注意Thread对象由系统控制，他们可以在应用进程之外进行修改。对于这个原因，需要在中断之前锁住线程访问，将访问放到synchronized块。比如：

public class PhotoManager {
	public static void cancelAll() {
	// create an array of Runnables that's the same size
	// as the thread pool work queue
	Runnable[] runnableArray = new Runnable[mDecodeWorkQueue.size()];
	// populates the array with the runnables in the queue
	mDecodeWorkQueue.toArray(runnableArray);
	// store the array length in order to iterate over the // array
	int len = runnableArray.length;

	// iterates over the array of runnables and interrupts // each one's thread
	synchronized(sInstance) {
		// Iterates over the array of tasks
		for (int runnableIndex = 0; runnableIndex < len; runnableIndex ++) {
			// gets the current thread
			Thread thread = runnableArray[taskArrayIndex].mThread;
			// if the thread exists, post an interrupt to it
			if (null != thread) {
				thread.interrupt();
			}
		}
	}
}

在大多数情况中，Thread.interrupt()立马中止线程。但是只能停止等待中的线程，不会中断CPU或网络密集的任务，避免减慢或锁住系统，需要在试图操作之前测试中断任务请求：

// 在继续运行之前，检查线程是否被中断
if (Thread.interrupted()) {
	return;
}

// decodes a byte array into a bitmap(cpu-intensive)
BitmapFactory.decodeByteArray(
	imageBuffer, 0, imageBuffer.length, bitmapOptions);