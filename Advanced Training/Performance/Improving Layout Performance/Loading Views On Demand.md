Loading Views On Demand 按需加载视图

有时候布局可能需要复杂的视图并且很少会用到。无论他们是item详情，进度指示器，或者撤销消息。当在需要用到他们的时候再去加载，这样做得好处是减少内存使用并且加速渲染。

Define a ViewStub 定义ViewStub

ViewStub是一种轻量级的视图，在布局中它没有尺寸并且不绘制和参与任何东西。所以，它初始化简单并且放置到视图层级中也很划算。每一个ViewStub简单地设置android:layout属性用于指定将要加载的布局。

下面的ViewSutb是用于一个透明的进度条遮罩。只有在新的item被导入到应用程序中可见。

<ViewStub
   android:id="@+id/stub_import"
   android:inflatedId="@+id/panel_import"
   android:layout="@layout/progress_overlay"
   android:layout_width="match_parent"
   android:layout_height="wrap_content"
   android:layout_gravity="bottom"/>

Load the ViewStub Layout 加载ViewStub布局

当想要加载ViewStub指定的布局时，既可以调用setVisibility(View.VISIBLE)设置可见，也可以调用inflate()方法。

((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);

或者

ViewimportPanel = ((ViewStub)findViewById(R.id.stub_imort)).inflate();

注意：inflate()方法在初始化完成后返回对应的View. 如果想要对view内部布局交互的时候，调用findViewById()

一旦 处于可见/初始化 后，ViewStub元素不再是视图层级的一部分。他会被初始化的布局所替代，根视图的ID是由ViewStub中android:inflatedId属性设置的ID所替代。(ViewStub中android:id指定的Id仅仅在ViewStub中得布局处于可见或者初始化之前有效)。

注意，ViewStub的一个缺点是在当前的目标布局中不支持<merge/>标记。