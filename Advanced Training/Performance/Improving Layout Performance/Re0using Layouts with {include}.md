Re-using Layouts with <include/> 使用<include/>标签复用布局

尽管Android提供了多种小而可复用的控件，但是有时候需要复用那些需要特殊布局的复杂组件。为了搞笑复用复杂的布局，使用<include/>和<merge/>标记
将当前布局嵌入到另外布局中。

复用布局文件是非常有力地方式允许创建可复用的复杂布局。比如，yes/no按钮面板，或者带文本的自定义进度条。它也意味着在应用程序中，将多处地方中相同的元素提取出来，独立管理，虽有添加到对应的布局中。所以，当创建通过自定义View的形式创建独立UI组件，这样做比复用布局要更容易些。

Create a Re-usable Layout 创建可复用的布局文件

如果已经知道想要复用的布局，创建新的XML文件并且定义布局。比如，下面是从G-Keenya codelab中摘出的例子定义了包含在每个activity中标题栏(titlebar.xml):

<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
android:layout_width="match_parent"
android:layout_height=“wrap_content”
android:background="@color/titlebar_bg">

	<ImageView
	android:layout_width="wrap_content"
	android:laout_height="wrap_content"
	android:src="@drawable/gafriclog"/>

</FrameLayout>

根视图应该准确地知道在每个布局中要我们要显示什么样的内容


Use the <include> Tag 使用<include>标记

在想要添加可复用组件的布局文件中，添加<include/> 标记。比如，下面是G-Kenya codelab中使用上面代码的例子：

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
android:orientation="vertical"
android:layout_width="match_parent"
android:layout_height="match_parent"
android:background="@color/app_bg"
android:gravity="center_horizontal">

<include layout="@layout/titlebar"/>

<TextView 
	android:layout_width="match_parent"
	android:layout_height="wrap_content"
	android:text="@string/hello"
	android:padding="10dp"/>

...

<LinearLayout/>

可以在<include/>标识中重写所引用布局根节点的所有布局属性。比如：

<include android:id="@+id/news_title"
	android:layout_width="match_parent"
	android:layout_height="match_parent"
	layout="@layout/title"/>

但是，如果想要在标记内重写布局属性，必须重写android:layout_heighth和android:layout_width以便于其他布局属性生效。


Use the <merge> Tag 使用<merge>标记

<merge/>标记适用于一个布局在另外布局中的情况，帮助消除视图层级中冗余的view。比如，如果主布局是垂直的LinearLayout,内部有两个连续视图，他们可以在多个布局中被复用，现在被复用的布局需要根视图。但是使用另外一个LinearLayout作为可复用布局的根节点，将导致一个垂直方向的LinearLayout中包含一个垂直方向的LinearLayout.嵌套的LinearLayout会降低UI性能。

避免包含这样一个荣誉的试图组，作为替代的使用<merge>元素作为被复用的布局根view。比如：
<merge xmlns:android="http://schemas.android.com/apk/res/android">

<Button
	android:layout_width="fill_parent"
	android:layout_height="wrap_content"
	android:text="@string/add"/>

<Button
	android:layout_width="fill_parent"
	android:layout_height="wrap_content"
	android:text="@string/add"/>
<merge/>

现在，在其他布局中使用它的时候，系统将在<include>标记处忽略<merge>元素并且直接放置两个按钮在布局中.

