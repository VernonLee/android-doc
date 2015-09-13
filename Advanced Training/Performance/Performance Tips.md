Performance Tips 性能贴士

这篇文档主要介绍一些细节上的优化，将这些组合到一起提高应用综合性能，但是这些改变不太可能形成惊人的性能影响。选择正确地算法和数据结构是首要考虑的，相关介绍在本篇文档中没有包含。使用文档中的技巧点作为综合代码练习，这样就可以慢慢养成高效率编码的习惯。

编写高效率代码的基本原则：
· 不要做不需要的工作
· 如果可以避免的话就不要分配内存

对Android应用进行细节优化，面对最棘手的问题之一是应用肯定会运行在多种类型硬件的设备中。不同版本的VM处理器运行速度不同。

为了确保应用性能适用大部分设备，确认代码对所有级别有效并且尽力的优化性能。


Avoid Creating Unnecessary Objects 避免创建不需要的对象

创建对象从来都不是免费的，每个线程分配池的临时对象可以使得分配更廉价，但是分配内存经常要比不分配内存代价大。

在应用中分配更多地对象，要定期强制垃圾回收器回收，在用户体验中创建一点“打嗝”. 在Android 2.3中引入并发垃圾回收器，但是不要的工作应经常避免。

因此，避免创建不需要的对象实例。下面是一些小的帮助：
· 如果有个方法返回String, 并且知道无论如何他总会被附加到StringBuffer中，改变签名和实现，让该功能直接做追加而不是建立临时的对象。
· 当从输入数据中提取字符串，尝试返回原始数据的substring，而不是创建一份复制。创建新的String对象，但是它共享char[]数据。(权衡是如果只是用原始输入的一小部分，但是无论如何在内存中都会保留它)

一种更激进的想法是将多维数组转换成并行的单维数组：
· int数组比Integer数组更好，但是这个推广的事实是两个并行的int数组比（int, int）数组对象更高效。这同样适用于基本类型的任意组合。
· 如果需要实现存储(Foo, Bar)对象的容器，尝试记住两个并行的Foo[]和Bar[]数组通常要比单个(Foo, Bar)数组更好。(这种情况例外是当设计一个API用于其他代码的访问。在这些案例中，它通常是更好地在小的速度上妥协，以达到良好的API设计。但是在内部代码中，尽可能地使其高效)

常言道，避免创建短期临时对象。创建更少的对象意味着垃圾回收期工作量减少，直接影响到用户体验。


Prefer Static Over Virtual 

如果不需要访问对象变量，让方法成为静态。调用将增速15%-20%左右。这也是良好的练习，因为可以从方法签名告诉调用该方法不能改变对象的状态。

Use Static Final For Constants

在类的顶部考虑下面的声明方式：
static int intVal = 42;
static String strVal = "hello, world";

编译器生成一个类初始化方法，称之为<clinit>, 他在类第一次使用的时候被执行。该方法存储值42到intVal, 从类文件中字符串常量表中为strVal提取引用。当这些随后被引用，可以通过变量查找访问。

使用“final”关键字提高事项
static final int intVal = 42;
static final String strVal = "hello,world";
类不再需要<clint>方法，因为常量已经被初始化到dex文件中。代码中引用intVal将直接使用值42,并且访问strVal将使用一个相对的便宜的“字符串常量”
指令而不是变量查找。

注意：此类优化仅仅适用于原始类型和String常量，而不是随意地引用类型。依旧，良好的练习是只要有可能的使用static final声明常量。


Avoid Internal Getters/Setters 避免内部Getters/Setters

在原生态语言中，如C++. 常见的操作是使用getters(i = getCount())而不是直接使用(i = mCount)访问变量。在C++中这是一个优秀的习惯并且经常在其他面向对象语言中使用，比如c#和Java,因为编译器通常内联访问，并且如果需要限制或调试变量访问，可以在任何时间添加代码。

但是，在Android中这是一个糟糕的注意。虚方法调用代价大，远远超过了实例变量查询。合理地遵循常见面向对象编程习惯并且在公共接口中包含getters和setters,但是在class内部，经过经常直接访问变量。

没有Jit, 直接访问变量大约比引用getter快3x倍速。有JIT(直接访问变量比访问本地更廉价)，直接访问变量大约比引用getter快7x倍速。

注意如果使用ProGuard,就可以两全其美，因为ProGuard提供内联访问。


Use Enhanced For Loop Syntax 使用高级的循环语法

高级for循环(比如“for-each”循环)可以用于实现Iterable接口和数组的容器。在集合内，迭代器使用接口调用hasNext()和next()方法。对于ArrayList,手写的循环统计大约3x倍速快(有无JIT)，但是对于其他集合，高级循环语句将直接等同于明确地迭代器使用。

下面是集中可选择的数组迭代方法：
static class Foo {
	int mSplat;
}

Foo[] mArray = ...

pubic void zero() {
	int sum = 0;
	for (int i = 0; i < mArray.length; ++i) {
		sum += mArray[i].mSplat;
	}
}

public void one() {
	int sum = 0;
	Foo[] localArray = mArray;
	int len = localArray.length;

	for (int i = 0; i < len; ++i) {
		sum += localArray[i].mSplat;
	}
}

public void two() {
	int sum = 0;
	for (Foo a : mArray) {
		sum += a.mSplat;
	}
}

zero()是最慢的，因为JIT还不能在每次迭代中优化获取数组长度。
one()是较快的。它将所有东西存入本地变量，避免查询。只有数组长度提升性能受益。
two()在没有JIT的设备中速度最快，很难和设备中包含JIT的one()方法区分。它使用在java1.5中引入的高级循环语法。

所以，默认使用高级for循环，但是对于ArrayList遍历，考虑使用手写的循环语句。

提示：在Josh Bloch's Effective Java, 第46想


Consider Package Instead of Private Access with Private Inner Classes

思考下面的定义：
public class Foo {
	private class Inner {
		void stuff() {
			Foo.this.doStuff(Foo.this.mValue);
		}
	}

	private int mValue;

	public void run() {
		Inner in = new Inner();
		mValue = 27;
		in.stuff();
	}

	private void doStuff(int value) {
		System.out.println("value is" + value);
	}
}

这里重要的是定义一个私有内部类(Foo$Inner)，直接访问私有方法和外部类的私有实例变量。这样做是允许的，代码打印“值是27”.
问题是VM考虑从Foo$Inner中直接访问Foo的私有成员的做法不允许，因为Foo和Foo$Inner诗不同的类，即使JAVA允许内部类访问外部类的私有成员方法。为了跨越这道坎，编译器生成一组假的方法：

/*package*/ static int Foo.access$100(Foo foo) {
	return foo.mValue;
}
/*package*/ static int Foo.access$200(Foo foo, int value) {
	foo.doStuff(value);
}

内部类在需要访问mValue变量或者在外部类调用doStuff()方法的时候调用这些静态方法。意味着上面的代码真的压缩那些通过附加方法访问成员变量。之前讨论过，访问方法要慢于直接访问变量，这是一个语言方言导致的不可见的性能影响。

如果遇到上面的情况，通过声明内部类变量和方法的访问层次从内部类提升到包访问，而不是私有访问。不幸的诗这也意味着变量可以被同一个包中得其他类直接访问，所以不能在公共API使用它。


Avoid Using Floating-Point 避免使用浮点数

在Android设备中浮点数的速度比整数慢2x倍。

在速度上，float和double在如今硬件设备上没有差距。在空间上，double是2x倍大。对于桌面机器，假设不考虑空间因素，应该倾向于double而不是float.

同样，即使对于整形，某些处理器have hardware multiply but lack hardware divide. 诸如此类的例子，整数部分和modulus操作是在软件中执行的-考虑设计hash表或做一些数学计算。


Know and Use the Libraries
典型的例子是String.indexOf()和相关APS，Dalvik替换为内部固有。相似的有，System.arrayCopy()大约比在nexus one（有JIT）中手写的代码块9x倍。

提示：Josh Block's Effective Java, item 47有相关介绍


Use Native Methods Carefully 

提示：Josh Block's Effective Java, item 54


Performance Myths 性能的传说

在没有JIT的设备中，调用方法的时候，参数是准确类型的要比接口类型的高效。（比如，方法中HashMap map 比 Map map快，即使两种map都是HashMap）.


