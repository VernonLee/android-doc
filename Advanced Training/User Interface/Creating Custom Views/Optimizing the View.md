Optimizing the View 优化View

既然已经有一个在状态间响应手势和过渡的良好设计的View，保证View运行流畅。为了避免UI在使用的时候卡顿和不流畅，保证动画始终保持在60帧/秒。

Do Less, Less Frequently 

为了加速View,从常规项中排除不是必须的并且频繁调用的代码。onDraw()启动流程，将给予最大的播放。尤其在onDraw()中排除分配，因为分配可能导致垃圾回收器卡顿。在初始化或动画之间的时候分配对象。永远不要在播放的动画的分配。

除了使得onDraw()精简之外，同样保证调用尽可能地不要太频繁。绝大数调用onDraw()的结果是调用invalidate()方法，所以消除invalidate()的不必要调用。

另外一种非常耗时的操作是遍历布局。View在任何时候调用requestLayout(), Android UI 系统需要遍历整个View层级去找到每个View应该有多大。如果出现测量冲突，可能需要多次遍历层级。UI设计者有时创建深层次的ViewGroup嵌套从而获得UI特性表现。这些深层次的View目录导致性能问题。所以View尽可能的层次浅一点。

如果有一个复杂的UI，考虑写个自定义ViewGroup实现布局。和内置的View不同，自定义View可以对子节点尺寸和形状做应用特定的假设，因此，避免遍历对子节点的测量计算。PieChart实例展示了如何继承ViewGroup。PieChart拥有子View,但是他从来不测量他们。相反，它根据自定义布局算法直接设置他们的尺寸。
