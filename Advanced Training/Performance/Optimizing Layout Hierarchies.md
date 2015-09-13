Optimizing Layout Hierarchies 优化布局层级

常见错误的观念是使用基本布局结构构成许多高效的布局。但是，添加的每个控件和布局需要被初始化，布局和绘制。比如，使用嵌套的LinearLayout生成非常深得视图层级。并且，嵌套的几个LinearLayout使用layout_weight参数尤其耗时，因为每个孩子需要测量两次。当布局是重复初始化的时候这个观点尤为重要，比如在ListView或GridView中使用。

在这节课中，将会学习如何使用Hierarchy Viewer 和 Layoutopt审查并且优化布局。

Inspect Your Layout 审查布局

Android SDK 中有个名称为Hierarchy View的工具，他可以在应用程序运行的时候帮助分析布局。使用它可以发现布局性能的瓶颈。

Hierarchy Viewer允许在连接的设备或模拟器中选择运行的进程，随后显示布局树结构。在每个块中交通灯代表测量，布局和绘制性能，帮助标出本质问题。

比如，表一中显示在ListView的Item的布局。该布局在左边显示较小尺寸的bitmap并且在右边包含两个垂直堆叠的文本。在布局将被初始化多次的时候尤为重要-比如这一个-被优化的性能好处将会是几倍的。

hierarchyviewer工具在<sdk>/tools/目录下。当打开Hierarchy Viewer显示一些可用设备的列表以及正在运行的组件。点击Load View Hierarchy查看选择组件的布局层级。比如，表2显示表一布局的具体信息：


Revise Your Layout 修改布局

在表2中，可以看到一个三级层级并且在子项中列出的一些问题。点击某一个小项，显示每个阶段处理耗时。很清楚的看到哪一个花费较长的时间测量，布局和渲染，我们要做的就是优化这些。

使用该布局渲染一个完整的列表item耗时如下：
· Measure: 0.977ms
· Layout: 0.167ms
· Draw: 2.717ms

在上面的例子中，由于使用了嵌套LinearLayout布局，所以性能上有所减慢, 将布局层级数目减少有助于提高性能，尽可能地浅而宽而不是窄而深。使用RelativeLayout作为根节可以满足这一要求。所以，在设计改变为使用RelativeLayout，可以看到布局层级变为两级。改变后的布局目录树如下：

新的渲染耗时:
· Measure 0.598ms
· Layout 0.110ms
· Draw: 2.146ms

有细微的提升，但是这只是单个item的耗时，将listview的所有item加起来，可想而知。

大部分这些时间不同是因为在LinearLayout中使用layout_weight导致的，layout_weight属性导致测量速度减慢。所以在使用的时候考虑是否真的需要使用它。


Use Lint 使用Lint工具

良好的习惯是经常运行lint工具，搜索可能的优化项。Lint替代Layoutopt工具并且有更高级的功能。比如：
· 使用compound drawables - 当LinearLayout中包含图片和文本是，更高效的做法是将他们处理成符合drawable.
· Merge root frame - 如果FrameLayout是布局的根节点，并且没有提供背景颜色和内边距等等，那么可以使用<merge>标签替代它，这样做更有效率。
· Useless leaf - 布局中没有子节点或背景颜色，可以被移除for a flatter并且更高效的布局层级。
· Useless parent - 如果布局中得子节点没有兄弟节点，不是ScrollView或根布局，并且没有背景颜色，可以将它移除并且直接将子节点放到他所在布局的父节点中。
· Deep Layout - 如果布局中有多级嵌套对性能的影响是很大德。考虑使用s水平布局比如RelativeLayout或GridLayout以便于提升性能。默认最大层级深度是10.

Lint的其他好处是已经整合到Android Studio中，无论何时编译程序，Lint都自动运行。也可以为指定构建运行lint审查，或者为所有variants运行。

在Android Studio中得File>Setting>Project Setting选项中管理审查明细和配置审查。审查配置页显示支持的审查项。

如图

Lint具有自动修复某些问题的能力，为其他问题提供建议以及直接跳转至问题代码。




