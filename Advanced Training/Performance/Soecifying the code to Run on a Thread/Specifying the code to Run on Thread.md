Specifying the Code to Run On a Thread 指定代码在线程中运行

这节课讲述如何实现Runnable类，该类在Runnable.run()方法中运行代码(独立的线程中). 可以传递Runnable对象给其他线程并且运行。一个或多个Runnable对象执行一个特别的操作，有时候称之为任务。

Thread和Runnable都是基本类，二者能力有限。但是，他们构成了强大的Android类，如HandlerThread, AsyncTask和IntentService。Thread和Runnable都是ThreadPoolExecutor的基础。该类自动管理线程和任务队列，并且可以同时运行多个线程。


Define a Class that implements Runnable 定义实现Runnable类

实现Runnable比较简单，比如：

public class PhotoDecodeRunnable implements Runnable {
	public void run() {
      // code you want to run on the thread goes here
	}
}


Implement the run() Method 实现run()方法

在类中，Runnable.run()方法包含被执行的代码。通常，在Runnable中允许任何东西。记住，Runnable不会运行在UI线程中，所以Runnable中不能直接修改UI对象，比如View对象。为了和UI线程沟通，必须使用Communicate with the UI Thread中讲解的方法。

在run()方法的开始，设置线程使用后台优先级，调用Process.setThreadPriority()，参数为THREAD_PRIORITY_BACKGROUD. 这种实现方式减少Runnable对象和UI线程间资源竞争。

调用Thread.currentThread()方法，在Runnable中存储Runnable对象的线程引用。

下面的代码指出如何启设置run()方法：

class PhotoDecodeRunnable implements Runnable {
	// defines the code to run for this task
	public void run() {
		// 将当期线程移入后台
		Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
		// 存储当前线程到PhotoTask实例，这样该实例
		// 可以中断线程
		mPotoTask.setImageDecodeThread(Thread.currentThread());
	}
}