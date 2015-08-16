Making the View Interactive 使得View可交互

绘制Ui仅仅是创建自定义View的一部分。除此之外，需要使得View对用户输入做出相应，从某种方式上更接近现实世界活动。对象应经常和现实对象扮演相同的角色。比如，图片不能立即弹出并且在其他地方重新显示？。因为现实中对象不会这么做，图片应该做得时从一处移到另一处。

用户同样对微妙的行为敏感或从接口中察觉，最好的做法是模仿现实世界的微妙之处。比如，当用户滑动UI对象，应在延迟运动的开始感觉到摩擦，并且在结束的时候减速缓冲。

这一节示范如何使用android提供的功能在自定义View中 添加这些仿真行为表现。


Handle Input Gestures 处理输入手势

和其他UI框架一样，Android支持输入事件模型。用户行为传递事件并触发回调，这些回调接口被重写以便处理当前应用是如何响应给用户。Android系统中大多数常见输入事件是触摸，它触发OnTouchEvent(android.view.MotionEvent).重写该方法处理事件：

@Overridepublic boolean onTouchEvent(MotionEvent event) {
	return super.onTouchEvent(event);
}

触摸事件本身不是非常有用。当前定义的手势交互有轻触，拉，推，滑动和缩放。为了转换触摸事件为手势行为，Android提供 GestureDector.

通过传递一个继承GestureDector.OnGestureListener的实例构造GestureDector. 如果只是处理很少的手势，可以继承GestureDetector.SimpleOnGestureListener替代GestureDector.OnGestureListener接口。比如，以下代码创建继承GestureDetector.SimpleOnGestureListener并且重写onDown(MotionEvent)方法。

class mListener extends GestureDetector.SimpleOnGestureListener {
	@Override
	public boolean onDown(MotionEvent e) {
	return true;
}
mDetector = new GestureDetector(PieChart.this.getContext(), new mListener());
}

不论是否使用GestureDetector.SimpleOnGestureListener,必须实现onDraw()方法并且返回true.这一步是必须的因为所有的手势都是以onDown()消息开始的。如果返回false,对于GestureDetector.SimpleOnGestureListener而言，系统假定忽略手势，GestureDetector.onGestureListener的其他方法不会被调用。只有在真的想要忽略全部手势的时候返回false.一旦实现GestureDetector.onGestureListener并且创建GestureDetector实例，那么就可以使用GestureDetector处理onTouchEvent()接受的事件。

@Override
public boolean onTouchEvent(MotionEvent event) {
	boolean result = mDetector.onTouchEvent(event);
	if (!result) {
		if (event.getAction() == MotionEvent.ACTION_UP) {
			stopScrolling();
			result = true;
		}
	}
	return result;
}

当传递Touchevent给onTouchEvent(), 不会被认为是手势的一部分，并且返回false。 随后使用自己定义的手势解析代码。


Create Physically Plausible Motion  创建合理的物理运动

手势是控制触屏强有力的方式，但是他们有悖常理并且很难记忆，除非产生合理地物理效果。这方面的一个很好的例子是滑动手势，用户在屏幕上快速地移动手指，然后抬起。UI在某个方向上快速移动，然后减慢，好似推出一个飞轮并且设置纺纱。

但是，模拟飞轮的感觉不重要。许多物理和数学需要获取正确工作的飞轮模型。好在，android提供帮助类帮助模拟这个或其他现象。Scroller类是处理flywheel-style fling手势的基本。

开始滑动，调用fling()with the starting velocity and the minimum and maximum x and y values of the fling. 对于滑动值
速度值，使用GestureDetector计算值：

@Overri
public boolean onFling(MotionEvent e1, MotionEvent e2,
	float velocityX, float velocity) {
	mScroller.fling(currentX, currentY, velociyx/SCALE,
		, velocityY/SCALE, minX, minY, maxX, maxY);
	postInvalidate();
}

注意：虽然GestureDetector计算速度是物理精确，血多开发者感觉使用该值使得滑动动画太快。通常有因子4和8除以x和y速度值。

调用fling()方法为滑动手势设置物理模型。毕竟，需要定期调用Scroller.computeScrollOffset()方法更新。Scroller.computeScrollOffset()方法当时通过读取当前时间和使用物理模型去计算X和Y位置 更新Scroller对象内部状态。调用getCurrX()和getCurrY()去获取这些值。

大多数View传递Scroller对象的x和y位置直接给scrollTo(). PieChart实例有一点不同：他们使用当前滑动y的位置去设置chart旋转角度。

if (!mScroller.isFinished()) {
	mScroller.computeScrollOffset();
	setPieRotation(mScroller.getCurrY());
}

Scrolller类计算滑动位置，但是他不会自动应用这些值给View. It's your responsibility to make sure you get and apply new coordinates often enough to make the scrolling animation look smooth.有两种实现方式：
· 经常在调用fling()后调用postInvalidate()方法，为了强制重绘。该技术请求在onDraw()中计算滑动便宜并且在滑动便宜改变后调用postInvalidate().

· 设置一个ValueAnimator对象设置滑动动画的时间，并且通过调用addUpdateListener()方法添加一个监听器去处理动画更新。

PieChart实例使用第二种实现方式。该方式设置起来比较复杂，但是他更接近于动画系统并且不会请求可能不必要的View更新。这种做法的缺陷是ValueAnimator在API level 11之前是不可用的，所以这种方法在低于Android 3.0之前版本中不能使用。

注意：ValueAnimator在API level 11之前是不可用得，但是也可以在低版本中使用。只需要在运行的时候检查当前API level, 并且在当前版本低于11的时候忽略这一调用。

 mScroller = new Scroller(getContext(), null, true);
       mScrollAnimator = ValueAnimator.ofFloat(0,1);
       mScrollAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
           @Override
           public void onAnimationUpdate(ValueAnimator valueAnimator) {
               if (!mScroller.isFinished()) {
                   mScroller.computeScrollOffset();
                   setPieRotation(mScroller.getCurrY());
               } else {
                   mScrollAnimator.cancel();
                   onScrollFinished();
               }
           }
       });


Make Your Transitions Smooth 让你的过度平滑起来

我们期望UI模型在不同状态过度的时候平滑一点。UI元素的淡入淡出替代显示与消失。动作平滑的开始和结束替代突然的启动和停止。在Android Property animation framework中，在Android 3.0的时候引入，是得平滑的过度容易些。

为了使用动画系统，无论什么影响View表现得特性发生改变。不要直接改变特性。相反的，使用ValueAnimatr去改变。在下面的实力中，修改当前PieChar中选中的圆饼部分导致整个图表旋转。这样选择指针处于选中部分的中心。ValueAnimator旋转改变一段耗时几百毫秒，而不是立即设置新的旋转值。

mAutoCenterAnimator = ObjectAnimator.ofInt(PieChart.this, "PieRotation", 0);
mAutoCenterAnimator.setIntValues(targetAngle);
mAutoCenterAnimator.setDuration(AUTOCENTER_ANIM_DURATION);
mAutoCenterAnimator.start();

如果想改变的值是基本视图属性之一，动画做起来很容易，因为有一个内置的ViewPropertyAnimator被用于优化多种同时发生的属性动画。比如：
animate().rotation(targetAngle).setDuration(ANIM_DURATION).start();













