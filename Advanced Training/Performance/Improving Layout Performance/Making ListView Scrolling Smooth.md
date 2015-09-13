Making ListView Scrolling Smooth ListView更光滑的滑动

让ListView滑动流畅的关键是保持应用程序主线程(UI线程)避免做一些耗时的操作。确保任何关于磁盘访问，网络访问，以及SQL访问都再独立的线程中进行。为了测试应用的状态，可以启用StrictMode测试。

Use a Background Thread 使用后台线程

使用后台线程(工作线程)减轻主线程的压力，让他专注于Ui绘制。在许多情况中使用AsyncTask提供一种简单地方式在主线程之外完成任务。AsyncTaks自动排列所有execute()请求并且有序执行。这种表现的目的是不用担心自己创建线程池。

在下面的实例代码中，AsyncTask用于在后台加载图片，完成后应用于UI。此外，在加载的过程中会显示一个进度条。

// Using an AsyncTask to load the slow images in a background thread

new AsyncTask<ViewHolder, Void, Bitmap>() {
	private ViewHolder v;

	protected Bitmap doInBackground(ViewHolder ... params) {
		v = params[0];
		return mFakeImageLoader.getImage();
	}

	protected void onPostExecute(Bitmap result) {
		if (v.position == postion) {
			// if this item hasn't been recycled 
			// already,hide the progress and set and show // the image
			v.progress.setVisibility(View.GONE);
			v.icon.setVisibility(View.VISIBILI);
			v.icon.setImageBitmap(bitmap);
		}
	}
}.execute(holder);

从Android 3.0(API 11)开始, AsyncTask可以带参数，所以可以启用它并且在多处理器内核间运行。替代调用execute()，可以指定executeOnExecute()并且多个请求可以同时被执行，具体数据根据处理器内核数量而定。


Hold View Objects in a View Holder 在ViewHolder中保管视图对象

在滑动ListView的过程中，可能会多次调用findViewById(),这样做降低性能。即使Adapter返回一个初始化的对象以便于循环，仍然需要查找元素并且更新他们。替代使用findViewById()的方式是使用“View Holder”设计模式。

ViewHolder对象存储布局中单项中所有的变量，这样可以立即访问他们而不是不停的查找。首先，创建一个类保管view集合，比如：

static class ViewHolder {
	TextView text;
	TextView timestamp;
	ImageView icon;
	ProgressBar progress;
	int position;
}

随后在布局中发布并存储ViewHolder

ViewHolder holder = new ViewHolder();
holder.icon = (TextView) converView.findViewById(R.id.listitem_image);
holder.text = (TextView) converView.findViewById(R.id.listitem_text);
holder.timestamp = (TextView) converView.findViewById(R.id.listitem_timestamp);
holder.progress = (ProgressBar) converView.findViewById(R.id.progress_spinner);
converView.setTag(holder);

现在很轻松的访问每一个View并且不需要查找，节省宝贵的处理器循环。