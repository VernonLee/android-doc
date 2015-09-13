Managing Your App's Memory 管理应用内存

随机访问内存（RAM）在任何软件开发环境中都是有宝贵的资源，但是在物理内存受限制的手机操作系统中更加宝贵。尽管android虚拟机调用垃圾回收期，但是这也不能意味着忽视应用在什么时候以及什么地方分配内存和释放内存。

为了使垃圾回收期回收应用内存，需要避免引入内存漏洞(通常由持有全局成员对象引起的)并且在合适的时候(有生命周期回调定义，后续具体讨论)释放任何Reference对象。对于大多数应用，虚拟机的垃圾回收期需要休息：系统在对应对象离开应用活动线程范围之外的时候回收分配的内存。

这篇文档解释Android如何管理应用进程和内存分配，以及在android开发的时候如何主动减少内存使用。更多关于在Java编程清除资源的练习，参考其他书本或关于管理资源引用的在线文档。如果正在寻找关于如何分析应用内存相关信息，阅读 Investigating Your RAM Usage.


How Android Manages Memory Android如何管理内存

Android不提供为内存交换空间，但是他使用paging和memory-mapping(mmapping)管理内存。这意味着任何内存修改-无论是通过分配新的对象或触摸mmapped页，仍驻留在内存中，不能被调用。所以只有一种方式去完全释放应用中得内存，释放可能引用的对象，使得内存对垃圾回收期可用。有一个例外：任何没有修改的mmapped文件，比如代码，如果系统想要在别处使用，可以调出RAM。

Sharing Memory 共享内存

为了适应需要在RAM的一切，Android试图跨进程共享RAM页。可以按照以下方式进行：
· 每个应用进程是从称之为Zygote的存在进程中分支出来。Zygote进程在当系统引导和加载常用框架代码和资源(比如activity主题)的时候启动。为了启动新进程，系统从Zygote分支处新应用进程，锁喉在该进程中加载和运行应用的代码。这就允许大多数为框架代码和资源分配的RAM页可以在所有应用进程之间共享。
· 大多数静态数据是映射到进程中。这不仅允许相同数据可以在不同进程间共享，而且允许根据需要调出。静态数据包含：Dalvik代码(为直接映射放置于预链接的 .odex文件)，应用资源(通过设计资源表结构，可以通过调整映射和ZIP apk的条目)，并且传统工程对象如静态代码存于.so文件。
· 在许多地方，Android跨进程共享同一个动态RAM，明确分配共享内存区域(和ashmem或gralloc一起使用)。比如，window surface在应用和屏幕图形合成器之间使用共享内存，光标缓存区在内容提供者和客户端之间使用共享内存。

由于广泛使用共享内存，考虑并确定应用中使用多少内存。关于确定应用内存使用讨论，详见 Investigating Your RAM Usage.

Allocating and Reclaiming App Memory 分配和回收应用内存

下面是一些关于Android如何分配和回收内存的实际情况:
· Dalvip 堆为每个进程限制单个虚拟机内存区域。定义逻辑堆尺寸大小，可以根据需要增长（但是不能超过系统为每个应用定义的最大值）。

· 堆的逻辑尺寸和堆使用的物理内存数量不一样。当检查应用的堆，Android计算一个名为 Proportional Set Size (PSS)比例集尺寸，这与其他进程共享既脏又干净的页-但仅在多少应用共享那一RAM的比例值。 PSS总量是应用物理内存空间考虑的。更多关于PSS的信息，查看Investigating Your RAM Usage指导。

·Dalvip 堆不会压缩逻辑尺寸，意味着Android不会支付堆堵塞空间。Android仅仅在没用的堆空间位于Dalvik堆末尾的时候减少逻辑堆尺寸。但是这不意味堆使用的物理内存不会减少。在垃圾回收器回收之后，Dalvik遍历堆并发现没用页，随后将这些页返回内核处理(madvise). 所以，大量块成对的分配和重新分配应最终回收所有(或附近所有)使用的物理存储。但是，从小量分配中回收内存效率较低，因为小量分配的页可能忍然被其他还没有释放的共享。

Restricting App Memory 限制应用内存

为了保持多任务功能环境，Android为每个应用堆内存设置硬性限制尺寸。不同设置该值的大小不同，具体根据设备总体有多少可用RAM。如果应用达到堆的最大限制并且尝试分配更多内存，将导致OutOfMemoryError.

在某些情况中，可能想要查询当前设备上系统可用的堆尺寸准确值-比如，决定在缓冲中保持多少数据是安全的。通过调用getMemoryClass()方法该值。他会返回一个整形值指出当前应用可用堆内存的M字节。更深层次的讨论在后续小节Check how much memory you should use中。

Switching Apps 切换应用

当用户在应用程序间切换的时候使用交换内存，Android在最近使用缓冲(LRU)中保持进程。比如，当用户首先加载一个应用，为他创建一个进程，但是当用户离开应用后，那个进程不会退出。系统保持进程缓冲，所以如果用户稍后返回应用的时候，该进程在应用切换回来的时候快速重用。

如果应用有缓冲进程并且保持当前不需要的内存，那么对于应用-即使当用户不在使用-限制系统综合性能。所以，在系统运行在低内存的时候，可能会杀掉LRU缓冲中的进程
开始最近使用的进程，但是会给与一定的考虑大多数内存密集的进程。为了保持进程缓冲尽可能的长久，参照下面章节给出的什么时候释放引用的建议。

更多关于进程不运行在前台的时候是如何缓冲的信息以及android设备是如何杀掉进程的信息，详见 Processes and Threads章节。


How Your App Should Manage Memory 应用该如何管理内存

在整个开发过程中应考虑到RAM的限制，包括在app设计(在开始开发之前).有许多种得到更高效结果的设计和编写代码的方式，通过聚合相同的技术一遍又一遍的应用。

Use services sparingly 谨慎使用services

如果应用需要在后台运行一个service的时候，不要一直让他运行除非激活它执行某个任务。Also be careful to never leak yout service by failing to stop it when its work is done.

当启动service,系统更倾向于为运行的service保留进程。这就使得进程非常昂贵，因为service使用的RAM不能被其他使用或调出页。导致系统保留在LRU缓存中缓存进程的数量减少，使得应用切换效率低下。甚至可以在内存紧张并且系统不能维持足够的进程去承载正在运行的服务的时候导致破坏系统。

最好的方式是限制service的寿命，替代使用IntentService, 他可以在处理启动它的Intent完成之后就结束自己。更多详细信息，阅读Running in a Background Service.

在不需要service运行的时候却没有制止是android应用最容易犯的最差内存管理错误之一。所以不要贪婪的保留service一直运行。不仅增加由于RAM限制而导致应用性能受阻的风险，而且在用户发现这一糟糕表现的时候很有可能卸载它。

Release memory when your user interface becomes hidden
当用户界面隐藏释放内存

当用户导航到另外一个应用，并且当前应用UI不再可见，需要释放UI使用的任何资源。这个时候释放UI资源显著增加系统缓存进程的能力，直接影响用户体验的质量。

当用户离开UI的时候得到通知，实现Activity类中onTrimMemory()回调。使用该方法监听TRIM_MEMORY_UI_HIDDEN事件，它指明UI隐藏并且释放UI使用的资源。

注意应用收到onTrimMemory()回调仅仅是在应用进程的所有UI组件隐藏起来。他和onStop()不同，后者是在当Activity实例影藏的时候调用，即使是用户跳转到其他应用。所以尽管需要实现onStop（）释放activity资源，比如网络连接或取消广播接收器的注册。通常不应该释放UI资源直到受到onTrimMemory(TRIM_MEMORY_UI_HIDDEN).这就确保如果用户从应用中其他页面返回当前页面的时候，UI资源仍然可以使用。


Release memory as memory becomes tight 当内存告急的时候释放内存

在应用生命周期的任何阶段，onTrimMemory()回调告知设备整体内存不足。应做响应并且根据以下内存级别释放资源。

· TRIM_MEMORY_RUNNING_MODERATE
应用正在运行并且没有考虑被杀掉，但是设备正运行在内存不足并且系统准备杀掉LRU缓存中得进程

· TRIM_MEMORY_RUNNING_LOW
应用正在运行并且没有考虑被杀掉，但是设备内存较少，所以需要释放不用的资源以提高系统性能(直接影响到应用性能).

· TRIM_MEMORY_RUNNING_CRITICAL
应用任然正在运行，但是系统已经杀掉了LRU缓存中大部分进程，所以现在需要释放不是很重要的资源。如果系统不能回收足够数量的RAM，将会清除所有LRU缓存并且开始杀掉那些系统倾向于保持的进程，比如正在运行的service

同样，当应用进程被缓存后，可能收到从onTrimMemory()中获取的消息中之一：

· TRIM_MEMORY_BACKGROUND
系统正运行在低内存设备并且进程靠近LRU列表的开始部分。虽然应用进程没有处于一个被杀掉的高风险状态，系统可能已经在LRU缓存中杀掉进程。需要释放那些容易恢复的资源，保证进程保持在列表中并且可以快速地回到应用。

· TRIM)MEMORY_MODETATE
系统运行在低内存设备并且进程位于LRU列表的中部分。如果系统对内存限制很严格，那么该进程很可能被杀掉。

· TRIM_MEMORY_COMPLETE
系统正运行在低内存设备并且在系统没有恢复内存的时候，该进程是首先被杀掉的其中之一。需要释放任何对应用状态不重要的东西。

应用onTrimMemory()回调在API Level 14中添加，在低版本设备上使用该回调，其效果等同于TRIM_MEMORY_COMPLETE事件。

注意：当系统开始杀掉LRU缓存中的进程时，虽然主要是从底部到顶部，但是还是会考虑那些进程消耗的内存更多并且如果杀掉可以为系统获取较多的内存。所以在LRU列表中整体上消耗内存较少，那么就可以保留在列表中从而快速回到应用。


Check how much memory you should use 检查要使用多少内存

在之前已近提到，每个android设备的可用RAM是不同的，因此每个应用的堆限制也不一样。调用getMemroyClass()获取粗略的应用可用堆内存。如果应用试图分配超过可用限制值的内存，将回收到OutOfMemoryError.

在非常特殊的情况下，通过设置manifest中<application>标记的largeHeap属性为true，获取较大堆尺寸。并且调用getLargeMemoryClass()获取该尺寸粗略值。

但是，请求一个大的堆尺寸仅仅适合一小部分需要消耗大量RAM的应用(比如，相片编辑应用)。切记不要随意的请求大尺寸，因为会耗尽内存并且需要快速修复-只有在准确知道所有内存在哪里被分配并且为什么必须保留。甚至，即使应用要求大尺寸，也要避免请求可能无论什么范围。使用额外的内存将综合决定用户体验，因为垃圾回收将耗时更长并且系统性能在任务切换或执行其他常用操作的时候变的很慢。

除此之外，较大的堆尺寸值因设备不同而不同，当运行在有RAM限制的设备上，较大的对尺寸可能和常规堆尺寸值相近。所以即使请求较大的堆尺寸，也要调用getMemoryClass()检查常规堆尺寸并且尽量保持低于限制值。

Avoid wasting memroy with bitmaps 避免bitmap浪费内存

当加载bitmap,仅在分辨率适用当前设备屏幕的时候保留到RAM中，如果原始图片分辨率较高，那么缩放到低一点的值。牢记增加bitmap分辨率最终导致响应的内存值增加，因为X和Y尺寸都增加。

注意：在Android 2.3（API Level 10）及低版本中，bitmap对象在应用堆中经常显示相同尺寸而不理会图片分辨率(实际像素数据独立保存在本地内存中). 这就使得很难去调试bitmap内存分配，因为大多数堆分析工具看不到本地分配。但是，在Android 3.0 (API 11)开始，bitmap像素数据被分配到应用Dalvik堆中，提高垃圾回收率和可调试度。所以，如果应用使用Bitmap并且发现应用在旧设备上遇到使用内存的麻烦，那么在切换到Android 3.0或更高版本中运行和调试。

更多关于使用bitmap信息，查看Managing Bitmap Memory


Use optimized data containers 使用优化的数据容器

利用Android框架提供的优化的容器, 比如SparseArray,SparseBooleanArray,和LongSparseArray. 普通HashMap实现效率较低，因为它需要为每个图搭配一个分离的实体对象。除此之外，SpaseArray类更加高效，因为他们避免the system's need to autobox the key and sometimes value(which creates yet another object or two per entry). and don't be afraid of dropping down to raw arrays when that makes sense.


Be aware of memory overhead 经常考虑内存

了解使用的语言和库的开支，在设计应用的时候，自始至终都要记住这一点。经常，表面上是无害的东西实际上有大量的开支。比如下面这些情况：

· 泛型请求的内存是静态常量的两倍。在android中使用泛型应当严格。
· java中每一个类(包含匿名内部类)代码大约使用500字节。
· 每个类的实例有12-16字节的RAM开支。
· 将单个实体放到HashMap中，要求分配一个额外的实体对象占用32字节

在这里或其他地方迅速增加的少量字节-应用设计类或笨重的对象将遭受这一开支。这就导致很难进行堆分析并且意识到问题来自于这些大量的小对象消耗完RAM。


Be careful with code abstractions 小心抽象

经常性的，开发者使用简单抽象作为“优秀编程练习”，因为抽象可以提高代码流畅度和可维护性。但是，抽象会带来显著消耗：通常他们要求更多的代码去被执行，要求更多时间以及为了将代码保存到内存中而需要更多的RAM。所以，如果抽象没有提供显著受益，避免使用他们。


Use nano protobufs for serialized data

Protocol buffers是语言中立，平台中立，谷歌为序列化结构数据设计的可扩展机制，和XML相似?, 但是更轻巧，更简单，更快速。如果决定使用protobufs，那么应该经常在客户端代码中使用nano protobufs。常规protobufs生成的代码非常啰嗦，并且在应用中导致各种类型的问题：增加RAM使用，APK尺寸增加，执行速度减慢，并且很快地超过DEX符号限制。

更多关于“Nano version”信息，查看protobuf readme小节。


Avoid dependency injection frameworks 避免依赖注入框架

使用依赖注入框架，比如Guice或RoboGuice很有吸引力，因为他们简化代码并且提供adaptive环境对测试或其他配置改变很有帮助。但是，这些框架趋向于通过扫描代码做一些进程初始化。这就要求显著数量的代码加到RAM，即使不需要这么做。这些mapped pages被分配到干净内存以至于Android可以丢弃他们，但是这种情况只有在pages在memory中被放置很长时间才会发生。


谨慎使用外部库

外部库代码经常不是为手机环境编写的并且在手机客户端使用时效率很低。极少数情况，当决定使用外部库，假设已近进行移植并且对该库优化。在决定使用它之前，预先分析库代码尺寸和RAM占用空间。

即使库据说被设计用在Android,这中本质上是危险的。因为每个库可能做得工作不一样。比如，某个库可能使用nano protobufs而其他使用micro protobufs。现在有两种不同protobuf实现方式。这种情况也可能发生在不同的日志工具，分析工具，图片加载框架，缓存，以及各种所不期望的东西。Proguard不会节约，因为这些都是被想要的库功能要求的低级别依赖。当使用库(趋向于宽泛的依赖)中Activity的子类时会变得尤为麻烦，当库使用反射(这种很常见并且意味着需要花费大量的时间手动配置ProGuard工作)，也是用一个道理。

另外，小心掉进包含许多功能的共享库而只用其中一两个的陷阱；不希望加入大量代码并且这些代码不会去用。在最后，如果没有一个和我们所需要的功能之间有很高的吻合的存在实现，最好的方法是创建自己的实现。


Optimize overall performance 优化综合性能

关于优化应用综合性能的信息在Best Practices for Performance中有列举。文中大部分介绍优化CPU性能技巧，但是许多技巧点同样适用于内存使用，比如减少要求UI中得布局对象数量。

一边阅读关于optimizing your UI，一边使用布局调试工具，充分利用lint tool提供的优化建议。


Use Proguard to strip out any unneeded code 使用Proguard去掉不需要的代码

ProGuard工具缩减，优化和混淆代码(移除无用的代码和使用很难理解的名称重命名类，变量和方法)。使用ProGuard帮助代码更健壮，请求更少RAM。


Use zipalign on your final apk 使用zipalign优化最终apk文件

如果对由Build system(包含数字签名和产品证书)生产的APK做任何post-processing处理，必须执行zipalign重写校准apk.操作失败导致应用要求更多的RAM，因为类似于资源等文件不再被mmapped.

注意：Google Play Store不接受没有经过zipaligned的apk文件


Analyze your RAM usage 分析RAM使用

一旦实现相对稳定的构建，开始分析应用在整个生命周期中使用多少RAM。更多关于如何分析应用，阅读Investigating Your RAM Usage.


Use multiple processes 使用多线程处理

如果有一种合适的方法用于应用，一种可能帮助管理应用内存高级的技巧点是将应用组件放到多线程。在使用的时候必须经常小心并且大多数应用不应该运行多线程，因为在错误的做法下很容易增加-而不是减少-RAM占用空间。主要用于那些可能和前台一样在后台运行大量工作的应用并且分开管理这些操作。

最合适的例子是在创建音乐播放器，播放音乐是在service中并且占用较长的时间。如果整个应用运行在单个线程中，那么为Activity UI执行的分配必须
和音乐播放一样长，即使用户当前在其他应用中并且service正在控制播放。像这样的应用可以分为两个进程：一个用于UI，另一个继续在后台播放服务。

在manifest文件中，在每个组件中声明 android:process属性指定分离的进程。比如，可以指定service运行于和应用主线程分开的进程，声明该进程名称为"background"（名称可以随意指定）

<service android=".PlayService"
		 android:process=":background"/>

进程名字必须以（":"）开头，确保该进程是当前应用私有的。

在决定创建新进程之前，需要理解内存可能的结果。为了说明每个进程的重要性，考虑一个空进程什么都不做大约占用1.4MB内存空间，memory information dump 详细信息如下：

如图

注意：更多关于如何阅读上面输出，在investigating Your RAM Usage中有提供。这里的关键数据是Private Dirty和Private Clean memory,大约使用接近1.4MB未分页RAM(分配跨越Dalvik heap, native allocations, book-keeping和library-loading),RAM的其他150K用于has been mapped in to execute的代码。

空进程的内存占用是相当显著的并且在该进程中开始工作的时候快速增长。比如，下面是创建一个只包含一些文字的Activity的内存使用情况:

如图

进程占用4M左右，在UI中简单显示一些文本。这导致一个严重的后果：如果将应用分为多线程，其中一个用于响应UI。另一个则避免任何UI，进程要求的RAM将快速增加(尤其一旦开始加载图片和其他资源)。在UI绘制后内存使用很难或者说不可能减少。

除此之外，当运行超过一个进程，尽可能地保持代码高效率，因为任何非必须的RAM为常见实现在每个进程中自我复制。比如，如果使用泛型(在you should use enums)，所有的RAM需要被创建和初始化这些变量在每个进程中是复制的，并且任何抽象you have with adapters and temporaries or other overhead will likewise be replicated.

多线程其他所牵扯的是他们之间的依赖。比如，应用的默认主进程中有一个内容提供者，在后台进程中添加使用内容提供者的代码，这将同样要求UI进程在RAM保留。如果目的是在后台进程中独立运行轻量级UI进程，那么它就不能依赖内容提供者或在主线程中得服务。总之一句话，线程之间最好不要有相互























































