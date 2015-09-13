Keeping Your App Responsive 保持应用响应

有这么一种情况，编写的代码通过世界上每一种性能测试，但是仍然感觉不流畅，偶尔出现卡顿，或者花费太长时间而不能输出输入。最糟糕的可能是应用突然出现“Application Not Responding” ANR 对话框。

在Android中，系统留意应用在一段时间内没有响应，就会显示对话框，提示应用已经停止响应。此类特征，由于应用一段时间没有响应所以系统提供用户选项退出应用。批判性的不要设计无法响应应用，这样系统从不显示ANR对话框。

这篇文档讲述android系统判定应用是否未响应并且提供应用保持响应的指导方法。


What Triggers ANR? 什么触发ANR?
通常，当应用不能响应用户输入的时候弹出ANR。比如，如果应用中某些I/O操作(频繁网络访问)在主线程进行，操作中得主线程处于锁定状态，所以系统不能处理用户输入事件。或者应用花费大量的时间构建精美in-memory结构或者在主线程中计算游戏下一次移动位置。确定这些操作的高效性是非常重要的，但是一些高效的代码仍然花费一定的时间。

应用中，对于花费较长时间的操作情况，不应该在主线程中完成这些工作，作为替代的创建工作线程完成大部分工作。这么做保持UI线程(循环接受用户交互事件)一直运行。因为这些线程通常在类级别中完成，可以作为类问题考虑响应速度。(和基本代码性能比较，从方法级别上考虑问题)。

在Android中，应用程序响度由Activity Manager 和 Window Manger系统服务观察。 在检测到以下情况的时候弹出ANR对话框：
· 在5秒内对输入事件无响应(比如键盘按下或屏幕触摸事件)
· BroadcastReceiver在10秒内没有执行结束。


How to Avoid ANRs 如何避免ANR

Android应用通常在默认的主线程中运行。意味着在主线程中执行的工作占用较长时间会触发ANR对话框，因为应用无法处理输入事件或Intent广播。

因此，任何运行在主线程中的方法做的工作尽可能少。特别的，在activity中得生命周期方法，比如onCreate()和onResume()中也尽可能少做。本质上耗时的操作，比如网络或数据库操作，亦或者耗时的计算任务，如改变位图尺寸等等。上面的提到的操作应该在工作线程中完成(如数据库操作，通过异步请求实现)。

大多数高效的方式使用AsyncTask类为耗时的操作创建工作线程。简单地继承AsyncTask并且实现doInBackground()方法执行任务。调用publishProgress()方法发布工作进度，引用onProgressUpdate()回调方法接收并更新进度。

private class DownloadFilesTask extends AysnTask<URL, Integer, Long> {
	// 完成耗时任务
	protected long doInBackground(URL ... urls) {
		int count = urls.length;
		long totalSize = 0;
		for (int i = 0; i < count; i++) {
			totalSize += Downloader.downloadFile(urls[i]);
			publishProgress((int) ((i / (float) count) * 100));
			// Escape early if cancel() is called
			if (isCancelled()) break;
		}
		return totalSize;
	}

	// this is called each time you call publishProgress()
	protected void onProgressUpdate(Integer ... progress) {
		setProgressPercent(progress[o]);
	}

	// this is called when doInBackground is finished
	protected void onPostExecute(long result) {
		showNotification("Downloaded" + result + "bytes");
	}
}

启动工作线程，创建实例调用execute()方法：

new DownloadFilesTask().execute(url1, url2, url3);

尽管这要比AsyncTask复杂，可能想替代创建自己的Thread或HandlerThread类。如果这样做，需要调用Process.setThreadPriority()方法并传递THREAD_PRIORITY_BACKGROUND参数设置线程优先级为“后台”级别。如果使用这种方式设置线程到低一点的优先级，那么线程可能仍然减慢应用，因为它默认的优先级别和UI线程一致。

如果实现Thread或HandlerThread, 确认主线程在等待工作线程完成的时候没有阻塞-必要调用Thread.wait()或者Thread.sleep()。在等待工作线程完成的过程中，主线程应为其他线程提供Handler并且在完成后发送结果。使用这种方法设计应用可保证应用保持响应因此避免由输入事件5秒超时导致的ANR对话框。

BroadcastReceiver执行时间指定的限制是广播接收器意味着：少量，后台中不关联的工作，比如保持设置或注册Notification. 主线程中其他方法的调用，应用程序应该避免耗时操作或者在广播接收器中做计算。但是相反的，使用工作线程完成这些密集任务，如果耗时的事件需要在广播中响应，那么应用程序可以启动IntentService完成这些事件。

提示：使用StrictMode帮助找到耗时的操作，比如在主线程中偶然添加的网络或者数据库操作等耗时操作。


Reinforce Responsiveness 强化响应度

通常，100到200毫秒是用户察觉迟钝的下限。下面是一些在避免ANR之外提供的一些技巧点帮助提高应用响应度：
· 在后台执行任务的时候，显示任务操作的进度(比如在UI中显示ProgressBar)。
· 特别对于游戏，在工作现场中计算移动位置。
· 如果应用有花费事件的初始化设置阶段，考虑显示一个飞入的屏幕或尽可能地渲染主界面，指明加载正在进行并且异步填充数据。在一些其他情况中，需要以某种方式指出任务正在进行中，以免用户察觉应用没有反应。
· 使用诸如Systrace和Traceview等性能工具找到应用中的响应瓶颈。








































