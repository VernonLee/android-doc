Communicating with the ui Thread 和主线程交互

在之前的课程中已经学会如何从ThreadPoolExecutor中管理线程中启动任务。最后一节克重讲述如何从任务中发送数据给UI线程。该功能允许任务在后台工作并且将结果返回给UI线程。比如图片。

每一个应用都有自己的线程运行UI对象，比如View对象；我们把这个线程称之为UI线程。只有运行在UI线程的对象才可以访问线程中的其他对象。由于线程池中线程所承载的任务不是在UI线程运行，所以他们没有访问UI对象的权限。为了将数据从后天传递给UI线程，使用一个运行在UI线程的Handler对象。


Define a Handler on the UI Thread 在UI线程中定义Handler

Handler是安卓系统框架中管理线程的一部分。Handler对象接收消息并处理。通常，为新线程创建Handler,但是也可以为已经存在的线程创建Handler并将他们关联起来。当把Handler连接到UI线程，处理消息的代码在UI线程中运行。

在创建线程池的类构造方法中初始化Handler对象，保存在全局变量中。通过Handler(Looper)构造方法连接到UI线程。该构造方法使用Looper对象，他是Android系统线程管理框架的另一部分。什么样的Looper实例决定Handler初始化结果，Handler运行在和Looper同一个线程中。比如：

private PhotoManager() {
	...
	// Defines a Handler object that's attached to the UI thread
	mHandler = new Handler(Looper.getMainLooper()) {
	...
}

在Handler内部，重写HandlerMessage()方法。Android系统在收到新消息的时候调用该方法，该线程对象关联的所有Handler对象接收到同一个消息。比如：

/**
*	handleMessage()defines the operations to perform 
*	when the handler receives a new Message to process.
*/
public void handleMessage(Message inputMessage) {
	// get the image task from the incoming Message object.
	PhotoTask photoTask = (PhotoTask) inputMessage.obj;
}

the next section shows how to tell the handler to move data.


Move Data from a Task to the UI Thread 将数据传递给UI线程

为了将数据从正在运行的后台线程传达给主线程中的对象，在任务对象中存储数据引用和UI对象并启动。下一步，传递任务对象和状态吗给Handler. 发送一个包含状态信息和任务对象给Handler.因为Handler正运行在UI线程中，所以他可以将数据给UI对象。

Store data in the task object 在任务对象中存储数据

比如，这里有一个Runnable, 运行在后台线程，解码Bitmap并存储在父对象PhotoTask中。Runnable同样存储状态码DECODE_STATE_COMPLETED.

// A class that decodes photo files into bitmap
class PhotoDecodeRunnable implements Runnable {
	...
	PhotoDecodeRunnable(PhotoTask downloadTask) {
		mPhotoTask = downloadTask;
	}
	...
	// gets the download byte array
	byte[] imageBuffer = mPhotoTask.getByteBuffer().
	...
	// run the code for this task
	public void run() {
	...
	// tries to decode the image buffer
	returnBitmap = BitmapFactory.decodeByteArray(
	imageBuffer,
	0,
	imageBuffer.length,
	bitmapOptions);
	...
	// sets the imageview bitmap
	mPhotoTask.setImage(returnBitmap);
	// reports a status of "completed"
	mPhotoTask.handleDecodesState(DECODE_STATE_COMPLETED);
	}
}

PhotoTask包含显示Bitmap的ImageView句柄。即使引用至Bitmap和ImageView都是在同一个对象中，也不能分配Bitmap给ImageView,因为当前正没有运行在UI线程。

下一步是发送状态给PhotoTask对象。

Send status up the object hierarchy

在层级中，PhotoTask是次高对象。它包含解码的数据引用和显示数据的View.它从PhotoDecodeRunnable中接受状态码并把他交给线程池和Handler:

public class PhotoTask {
	...
	// Gets a handle to the object that creates the thread pools
	sPhotoManager = PhotoManager.getInstance();
	...
	public void handleDecodeState(int state) {
		int outState;
		// converts the decode state to the overall state
		switch(state) {
		 case PhotoDecodeRunnable.DECODE_STATE_COMPLETED:
		 	outState = PhotoManager.TASK_COMPLETED;
		 	break;
		 	...
		}
		...
		// call the generalized state method
		handleState(outState);
	}
	...
	// passes the state to photoManager
	void handleState(int state) {
	 // passes a handle to this task and the
	 // current state to the class that created the thread
	 // pools
	 sPhotoManager.handleState(this, state);
	}
}


Move data to the UI 传递数据

在PhotoTask对象中，PhotoManager对象接收状态码和PhotoTask对象句柄。由于状态是TASK_COMPLETED,创建包含状态和任务对象的Message并发送给Handler:

public class PhotoManager {
	...
	// Handle status message from tasks
	public void handleState(PhotoTask photoTask, int state) {
	switch(state) {
		...
		// the task finished downloading and decodeing the image
		case TASK_COMPLETED:
		// create a message for the handler with the state
		and the task object
		Message completeMessage = mHandler.obtainMessage(state, photoTask);
		completeMessage.sendToTarget();
		break;
	}
}
}

最后，Handler.handleMessage()为每一个收到的Message检查状态码。如果是TASK_COMPLETED,而且Message同时包含Bitmap和ImageView，任务结束。因为Handler.handleMessage()运行在UI线程，他可以安全的设置ImageView的图片。

private PhotoManager() {
	..
	mHandler = new Handler(Looper.getMainLooer()) {
		public void handleMessage(Message inputMessage) {
			// get the task from the incoming Message object.
			PhotoTask photoTask = (PhotoTask)inputMessage.obj;
			// Get the imageview from this task
			PhotoView localView = photoTask.getPhotoView();
			...
			switch(inputMessage.what()) {
				...
				case TASK_COMPLETE:
				// Move the bitmap from the task to the view
				localView.setImageBitmap(photoTask.getImage());
				break;
			 	default:
			 	// pass along other message from the UI
			 	super.handlerMessage(inputMessage);
			}
		}
	}
}
