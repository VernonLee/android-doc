Improving Layout Performance 提高布局性能

布局是Android应用程序中关键部分，它直接影响到用户体验。如果实现糟糕，布局导致内存紧张并且减缓UI流畅度. Android SDK中包含帮助标识布局性能问题的工具，分为以下小节，阅读全部章节后，就可以创建出光滑的界面并且花费最小内存占用。

Lessons 章节列表

Optimizing Layout Hierarchies 优化布局层级

相同的道理，复杂的WEB页减缓加载时间，如果布局层级也是很复杂，也会导致性能问题。这节课讲述如何使用SDK工具审查布局并且发现性能瓶颈。


Re-using Layouts with <include/> 使用<include/>标签复用布局

如果应用程序UI在很多地方重复布局结构，这节课讲述如何创建高效，可复用的布局结构，并且在适合的UI布局中包含他们。


Loading Views On Demand  根据需要加载视图

除了在布局中引用其他布局，还可能想要在需要显示的时候才可见，有时在activity运行之后。这一节讲述如何在需要的时候加载布局从而提高
布局初始化性能。


Making ListView Scrolling Smooth ListView光滑的滑动

如果使用ListView, 其中在每个单项中包含复杂或大量数据的内容，那么滑动性能可能会变差。这一节提供一些关于如何使得滑动性能更光滑的技巧。