Custom Drawing 重绘

自定义View最重要的部分是他的外观。重绘过程的难易程度根据应用所需决定。这一节包括一些最常见的操作。


重写onDraw()方法

在绘制View的重要的步骤之一是重写onDraw()方法。onDraw()的参数是用于绘制本身的Canvas对象。Canvas类可用于绘制文本，线条，图片以及一些其他基本图形。在onDraw()使用这些方法创建自己的自定义图形。

在调用任何绘制方法之前，必须先创建一个Paint对象。下一节具体讨论Paint。


Create Drawing Objects 创建绘制对象

android.graphics框架分割绘制过程为两个区域：
· 绘制什么，由Canvas处理
· 怎样去绘制，有Paint处理

比如，Canvas提供一个绘制线条的方法，Paint 提供定义线条颜色的方法。Canvas有一个绘制矩形的方法，Paint决定是否为该矩形提供填充色或保留为空。简单一点，Canvas定义在屏幕上绘制什么形状，Paint定义颜色，样式，字体以及绘制的形状。

因此，在绘制之前，需要创建一个或多个Paint对象。 PieChart实例在init方法中坐了该工作，并且在构造方法中调用：

private void init() {
	mTextPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
	mTextPaint.setColor(mTextColor);
	if (mTextHeight == 0) {
		mTextHeight = mTextPaint.getTextSize();
	} else {
		mTextPaint.setTextSize(mTextHeight);
	}

	mPiePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
	mPiePaint.setStyle(Paint.Style.FILL);
	mPiePaint.setTextSize(mTextHeight);

	mShowPaint = new Paint(0);
	mShowPaint.setColor(0xff101010);
	mShowPaint.setMaskFilter(new BlurMaskFilter(
		8, BlurMaskFilter.Blur.NORMAL));
	...
}

提前创建对象是个重要的举措。视图重绘非常频繁，并且许多绘制对象需要非常耗时的初始化。在onDraw()方法中创建绘制对象会显著减少性能并且会使得View显示非常呆滞。


Handle Layout Events 处理布局事件


为了准确绘制自定义的View,需要知道它的尺寸。复杂一点的自定义View经常需要根据屏幕上View区域的尺寸和形状执行多个布局计算。不要对屏幕上关于View的尺寸进行估测。即使只有一个应用使用该View，这就需要应用处理不同的屏幕尺寸，多种像素密度的屏幕以及在竖直和水平模式中各个方面的比值。

虽然View有很多处理测量的方法，其中大多数不需要被重写。如果View不需要特别的控制尺寸，那么重写onSizeChanged()就可以。

onSizeChanged()在View第一次分配尺寸的时候调用，当View尺寸在任何情况下发生改变的时候重写调用。onSizeChanged()方法负责计算位置，尺寸以及其他和View尺寸相关的值，而不是每次在绘制View的时候重写计算。在PieChart实例中，onSizeChanged()计算pie chart的矩形边界以及文本标签和其他可见元素的相对位置。

当View分配尺寸后，布局管理器假定View所有的边界尺寸。当计算View尺寸的时候必须处理这些边界值。以下是PieChart中如何处理的代码片段：

// Account for padding
float xpad = (float)(getPaddingLeft() + getPaddingRight());
float ypad = (float)(getPaddingTop() + getPaddingBottom());

// Account for the label
if (mShowText) xpad += mTextWidth;

float ww = (float)w - xpad;
float hh = (float)h - ypad;

// Figure out how big we can make the pie
float diameter = Math.min(ww, hh);

如果需要更好地控制布局参数，实现onMeasure()方法。该方法参数是View.MeasureSpec，该值指出View的父布局允许自定义的View有多大以及尺寸值是一个硬性最大值或仅仅是个建议值。作为优化项，这些值以打包好的整数存储，使用静态方法View.MeasureSpec解压存储的每个整数值。

下面是实现onMeasure()方法的实例。在该实现中，PieChart试图使区域足够大并且和标签一样大：

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
   // Try for a width based on our minimum
   int minw = getPaddingLeft() + getPaddingRight() + getSuggestedMinimumWidth();
   int w = resolveSizeAndState(minw, widthMeasureSpec, 1);

   // Whatever the width ends up being, ask for a height that would let the pie
   // get as big as it can
   int minh = MeasureSpec.getSize(w) - (int)mTextWidth + getPaddingBottom() + getPaddingTop();
   int h = resolveSizeAndState(MeasureSpec.getSize(w) - (int)mTextWidth, heightMeasureSpec, 0);

   setMeasuredDimension(w, h);
}

代码中注意三点：
· 在计算的时候将View的间距考虑到了。在之前提到过，这是View的责任。
· 帮助类方法resolveSizeAndState()用于创建最终高度和宽度值。此方法通过对比期望的尺寸和onMeasure()传递的指定值返回合适的View.MesaureSpec值。
· onMeasure()没有返回值。相反，通过调用setMeasuredDimension()方法传递结果。强制调用该方法。如果忽略这一操作，View类抛出运行时错误。


Draw! 开始绘制吧

一旦创建对象和测量的代码定义，就可实现onDraw()方法。每个View实现onDraw()方法不尽相同，但是大多数View都有一些通用操作：
· 使用drawText()方法绘制文本。调用setTypeface()方法指定字体，并且使用setColor()设置字体颜色。
· 使用drawRect(),drawOval()和drawArc()等方法绘制基本图形。调用setStlye()设置是否填充形状，轮廓以及全部。
· 使用Path类绘制较复杂的形状。为Path对象添加线条和曲线来定义形状，随后使用drawPath()方法绘制形状。和基本图形处理一样，调用setStyle()设置填充模式。
· 创建LinearGradient对象定义渐变填充。调用setShader()方法使用LinearGradient作用于填充的形状。
· 使用drawBitmap()绘制图片

比如，下面是绘制PieChart的代码。其中使用混合文本，线条和形状

protected void onDraw(Canvas canvas) {
	super.onDraw(canvas);

	// Draw the shadow
	canvas.drawOval(
		mShadowBounds,
		mShasowPaint);

	// Draw the label text
	canvas.drawText(mDta.get(mCurrentItem).mLabel,
	    mTextX, mTextY, mTextPaint);

	// Draw the pie slices
	for (int i = 0; i < mData.size(); ++i) {
		Item ite = mData.get(i);
		mPiePaint.setShader(ite.mShader);
		canvas.drawArc(mBounds,
			360 - it.mEndAngle,
			it.mEndAngle - it.mStartAngle,
			true, mPiePaint);
	}
	
	// Draw the pointer
	canvas.drawLine(mTextX, mPointerY, mPointerX, mPointerY, mTextPaint);
	canvas.drawCircle(mPointX, mPointY, mPointerSize, mTextPaint);
}

