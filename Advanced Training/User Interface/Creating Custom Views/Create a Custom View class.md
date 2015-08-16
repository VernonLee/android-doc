Creating a View Class 创建View类

设计良好的自定View是和其他设计良好的类相似。它囊括了易用的用户接口的实用集合，高效利用CPU和内存，并且物超所值。除此之外，对于一个设计良好的类，尽管是自定义View，按照以下要求：

· 符合Android标准
· 提供可用于XML布局的自定义样式属性
· 发送可达事件
· 兼容多种Android平台

Android框架提供一些类基本类和XML标记帮助创建满足这些要求。这一节讨论如何使用Android框架创建View类的核心功能。

View的子类

Android框架中所有View类都是继承View. 自定义View也可以直接继承View，或者为了节省事件直接继承定义好的View,比如按钮。

为了允许Android Developer Tools和创建的View交互，必须至少提供一个包含Context 和 AttributeSet参数的构造方法。该构造器允许布局编辑器创建和编辑View的实例。

class PieChart extends View {
	public PieChart(Context context, AttributeSet attrs {
		super(context, attrs);
	}
}

Define Custom Attrubutes 定义属性

为了添加内建的View到用户界面，在XML元素中指定并且使用元素属性控制外形和表现。好的自定义View可以通过XML添加和美化。在自定义View中启用这些效果，必须：
· 在<declare-styleable>资源元素中定义View的自定义属性。
· 在XML布局中为这些属性指定对应的值。
· 在运行的时候获取属性
· 在View中应用获取到得属性
这一节讨论如何定义自定义属性并指定他们的值。下一节处理在运行的时候获取和应用属性值。

为了定义自定义属性，为工程添加添加<declare-styleable>资源。通常将这些资源添加到res/values/attrs.xml文件中。下面是attrs.xml文件的实例：
<resources>
	<declare-styleable name="PieChart">
	<attr name="showText" format="boolean"/>
	<attr name="labelPosition" format="enum">
		<enum name="left" value="0"/>
		<enum name="right" value="1"/>
	</attr>
	</declare-styleable>
</resources>

这些代码声明两个自定义属性，showText 和 labelPosition,他们属于名称为 PieChart的样式实体。样式实体的名称是，按照惯例，和自定义View的类名移植。尽管没必要严格遵守这一规格，许多流行的代码编辑器都是按照这种规范提供语句完成。

一旦定义了自定义属性，可以类似和内建的属性一样在XML布局中使用他们。唯一的不同是自定义属性属于一个不同的命名空间。作为替代 http://schemas.android.com/apk/res/android命名空间，他们属于http://schemas.android.com/apk/res/[your package name]. 比如，下面是如何使用PieChart定义的属性：

<?xml version="1.0" encoding="utf-8">
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
 xmlns:custom="http://schemas.android.com/apk/res/com.example.customviews">
 <com.example.customviews.charting.PieChart
 	custom:showText="true"
 	custom:labelPosition="left"/>
 </LinearLayout>

 为了避免重复长的命名空间URL，实例中使用了xmlns命令。这一命令指派了别名custom给命名空间http://schemas.android.com/apk/res/com.example.customviews. 命名空间可以使用任何别名。

 注意添加到布局中自定义View标记。他是自定义View类的全地址名称。如果View类是内部类，必须在地址名称后面加上外部类的名称。比如，PieChart类有一个名称为PieView的内部类。为该类使用自定义属性，那么应该使用的标记为com.example.customviews.charting.PieChart$PieView

 使用自定义属性

 当在XML布局文件中创建View,XML标记中得所有属性都是从资源bundle中去读并且以AttributeSet传递到View得构造器。虽然可以直接从AttributeSet中直接读取值，但是这样做有很多缺点：
 · 属性值中的资源引用不能被解析
 · 样式不能被使用

 相反的，传递AttributeSet 到 obtainStyleAttributes(). 这一方法回传一个dereferenced和styled的TypeArray数组。

 为了使得调用obtainStyledAttributes()更容易写，android资源编译器已经做了一些工作。对资源目录中的每一个<declare-styleable>，生成的R.java文件定义了包含属性ids的数组和数组中为每个属性定义索引的常量集合。使用预定义常量从TypedArray中读取属性。下面是PieChart累如何读取属性：

 public PieChart (Context context, AttributeSet attrs) {
  super (context, attrs);
  TypedArray a = context.getTheme().obtainStyledAttributes (
  	attrs,
  	R.styleable.PieChart,
  	0, 0);

  	try {
  		mShowText = a.getBoolean(R.styleable.PieChart_showText, false);
  		mTextPos = a.getInteger(R.styleable.PieChart_labelPosition, 0);
  	} finally {
  		a.recycle();
  	}
  }
}

注意TypedArray对象共享资源，所以在使用后必须回收。

Add Properties and Events 添加特性和事件

属性是控制View的表现和外观强有力的方式，但是他们也仅仅在View初始化的时候被读取。为了提供动态的表现，通过getter和setter方法暴露每个自定义属性。以下代码片段显示如何PieChart暴露showText属性：

public boolean isShowText() {
	return mShowText;
}

public void setShowText(boolean showText) {
	mShowText = showText;
	invalidate();
	requestLayout();
}

注意 setShowText调用invalidate()和requestLayout()方法。这一举动十分重要确保View准确地运转。必须在任何可能引起View表现的改变之后invalidate view. 这样系统就知道是否需要重绘。同样地，如果某个可能影响View尺寸或形状的特性发生改变，那么就需要重新请求新的布局。牢记这些，避免引起难以找到的BUG。

自定义View同样应支持事件监听和重要的事件交流。比如，PieChart暴露一个 当用户旋转饼状图并将焦点置于新Pie部分的时候通知监听器的onCurrentItemChanged的自定义事件。

很容易忘记暴露特性和事件，尤其当很少人使用自定义View的时候。花费些事件在定义View接口上减少将来维护花费。一个良好的准则是经常暴露每个影响自定义View的可见的表现和外观的特性。

Design For Accessibility 为无障碍设计？

自定义View应该支持大量的用户。包含残疾人用户，防止他们看到或使用触摸屏。为了支持残疾人使用，需要：
· 使用 android:contentDescription属性为输入区添加提示
· 在合适的时候通过调用sendAccessibilityEvent()发送 accessibility事件.
· 支持可选的控制器，比如D-pad和轨迹球。
更多关于创建容易理解的View的信息，在开发者指导中查看Making Applications Accessible。
