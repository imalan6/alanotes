# JVM 类加载机制

## 概念

首先，在代码编译后，就会生成 JVM（Java 虚拟机）能够识别的二进制字节流文件（`*.class`）。**JVM 把 class 文件中的类描述数据从文件加载到内存，并对数据进行校验、转换解析、初始化，使这些数据最终成为可以被 JVM 直接使用的 Java 类型，这个过程就叫做 JVM 的类加载机制**。

类的加载指的是将类的`.class`文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个`java.lang.Class`对象，用来封装类在方法区内的数据结构。类加载的最终结果是位于堆区中的`Class`对象，`Class`对象封装了类在方法区内的数据结构，并且向 Java 程序员提供了访问方法区内的数据结构的接口。

![1.png](https://i.loli.net/2021/03/10/3cP2lVYH58nxmtX.png)

类加载器并不需要等到某个类被首次使用时再加载它，JVM 规范允许类加载器在预料某个类将要被使用时就预先加载它，如果在预先加载的过程中遇到了`.class`文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误（`LinkageError`错误）。如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误。

- **加载 .class 文件的主要方式如下：**

1）从本地系统中的`.class`文件直接加载

2）从`zip`，`jar`，`war`等归档文件中加载`.class`文件

3）通过网络下载`.class`文件

4）从专有数据库中提取`.class`文件

5）将 Java 源文件动态编译为`.class`文件

## 类的生命周期

类的生命周期是指一个 class 从加载到内存直至卸载出内存的过程，包含**加载**（Loading）、**验证**（Verification）、**准备**（Preparation）、**解析**（Resolution）、**初始化**（Initialization）、**使用**（Using）和**卸载**（Unloading）7个阶段，如下图所示：

![2.png](https://i.loli.net/2021/03/10/WJobdgj9qR4Bwe1.png)

其中，**验证、准备、解析三个阶段统称为连接（Linking），而加载、连接、初始化又可以统称为类加载的过程**，所以我们有时又可以称类的生命周期包含加载、连接、初始化、使用和卸载这5个阶段，或者是类加载、使用、卸载这3个阶段。

回到上图，加载、验证、准备、初始化和卸载这5个阶段的开始顺序是确定的，如图中箭头所示。之所以强调“开始顺序”，是因为这里的先后顺序仅仅是各阶段开始时间的顺序，而不是进行或完成的顺序，这些阶段通常是相互交叉地混合式进行的。比如加载和验证，并不是说非要等到加载完成之后，才开始验证阶段，在加载的阶段中，会穿插各种检验动作，否则对于连格式都不符合的字节流，又怎能正确解析出其中的静态数据结构从而转化为方法区中的数据结构呢？对于解析阶段，其开始时间则比较特殊，既可能在加载阶段就开始（对常量池中的符号引用的解析），也可能在初始化阶段之后才开始（支持 Java 语言的动态绑定）。

## 加载

- **主要流程**

加载过程不是指的类加载机制，而是类加载机制中的加载阶段。在这个阶段，JVM 主要完成3件事：

1）通过一个类的全限定名（包名 + 类名）来获取定义此类的二进制字节流（`class`文件）。

2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。方法区就是用来存放已被加载的类信息，常量，静态变量，编译后的代码的运行时内存区域。

3）在内存中生成一个代表这个类的`java.lang.Class`对象，作为方法区这个类的各种数据的访问入口。

相对于类加载机制的其他阶段，加载阶段相对来说可控性比较强，该阶段既可以使用系统提供的类加载器完成，也可以由用户自定义的类加载器来完成。开发人员可以通过定义自己的类加载器去控制字节流的获取方式。加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区中，而且在 Java 堆（永久代）中也创建一个`java.lang.Class`类的对象，这样便可以通过该对象访问方法区中的这些数据。

对于上述中所说的“内存”，虚拟机规范并没有明确规定是在 Java 堆还是方法区中，对于我们最为熟悉的 HotSpot 虚拟机，是存放在 Java 堆的永久代中。实际上永久代是 HotSpot 虚拟机特有的，是它对虚拟机规范中方法区概念的具体实现（JDK1.7 及以下，在 JDK1.8 及以上叫做元空间 metaspace）。

- **加载时机**

虚拟机规范中并未强制规定加载阶段具体什么时候开始，由虚拟机自由把握。就我们所熟知的 HotSpot 虚拟机来说，有两种情况：

1）预加载。虚拟机在启动时会预先加载`rt.jar`中的`class`文件，其中包括`java.lang.*`、`java.util.*`、`java.io.*`等运行时常用的类。

2）运行时加载。当虚拟机在运行过程中需要某个类时，如果该类的`class`未被加载则加载。

## 连接阶段

### 1、验证

连接的第一个阶段，**确保从 class 文件中所加载的字节流符合当前虚拟机的要求，且不会危害虚拟机自身的安全**。该阶段会依次进行如下验证：

1）文件格式验证：判断当前字节流是否符合`class`文件格式的规范。如是否以`class`文件的`oxCAFEBABE`开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中常量的类型是否合法等等。校验的目的是保证字节流能正确地解析并存储于方法区内，通过验证后，会在方法区中存储，后面的校验动作都是基于方法区的存储结构进行，不再直接操作字节流。

2）元数据验证：语义分析，判断其描述的信息是否符合 Java 语言的规范要求。如该类除了`java.lang.Object`之外，是否有其他父类；该类的父类是否继承了不允许被继承的`final`类等

3）字节码验证：通过数据流和控制流分析，判断程序语义是否合法、符合逻辑。如保证跳转指令不会跳转到方法体以外的字节码指令上、方法体中的类型转换是有效的等。

4）符号引用验证：发生在解析阶段将符号引用转为直接引用的时候，确保解析动作能正确执行。如符号引用中通过字符串描述的全限定名是否能找到对应类。

从上面可以看出，验证阶段非常重要，关乎虚拟机的安全，但它并不是必须的，它对程序运行期没有影响，如果所引用的类已被反复使用和验证过，那么可以考虑采用`-Xverifynone`参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。通常来讲，应用所加载的`class`文件都是由我们本地或服务器的 JDK 编译通过的，我们都确定它是符合虚拟机要求的，对于这类`class`文件其实并不需要验证，主要是像从网络加载的`class`字节流或是通过动态字节码技术生成的字节流，出于安全的考虑，是必须要经过严格验证的。

### 2、准备

准备阶段做的唯一一件事就**是为类的静态变量分配内存，并将其初始化为默认值**。注意这里的初始化和后面要讲的“初始化阶段”是不同的，容易混淆。这些内存都在方法区中分配。注意事项如下：

1）对于初始化为默认值这一点，有两个角度的理解：从 Java 应用层面讲，会为不同的类型设置对应的零值，如对于`int`、`long`、`byte`等整数对应就是 0，对于`float`、`double`等浮点数则是 0.0，而对于引用类型则是 null，有个零值映射表；从 JVM 层面，实际上就是分配了一块全 0 值的内存，只是不同的数据类型对于 0 值有不同的解释含义，这是 Java 编译器自动完成的。

2）如果类的静态变量是 final 类型的，即它的字段属性表中存在`ConstantValue`属性，那么在准备阶段就会被初始化为程序指定的值，比如对于`public static final int len = 10`，在准备阶段`len`的值已经被设置为10了。实际上对于`final`的类变量，在编译时就已经将其结果放入了调用它的类的常量池中，这种类变量的访问并不会触发其所属类的初始化阶段。

### 3、解析

该阶段把类在常量池中的符号引用转为直接引用。符号引用就是一组用来描述目标的字面量，说白了就是静态的占位符，与内存布局无关，而直接引用则是运行时的，是指内存中直接指向目标的指针、相对偏移量或间接定位到目标的句柄。解析工作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用限定符这7类符号引用，将其替换为直接引用。

## 初始化

- **类初始化方式**

初始化，就是执行类的构造器方法`init`()，为类的静态变量赋予正确的初始值的过程。JVM 负责对类进行初始化，主要对类变量进行初始化。在 Java 中对类变量进行初始值设定有两种方式：

1）声明类变量时指定初始值；

2）使用静态代码块为类变量指定初始值。

- **类初始化步骤**

1）假如这个类还没有被加载和连接，则程序先加载并连接该类；

2）假如该类的直接父类还没有被初始化，则先初始化其直接父类；

3）假如类中有初始化语句，则系统依次执行这些初始化语句。

- **类初始化时机**

只有当对类主动使用的时候才会导致类的初始化，`Java`虚拟机规范中严格规定了有且只有五种情况必须对类进行初始化：

1）使用`new`字节码指令创建类的实例，或者使用`getstatic`、`putstatic`读取或设置一个静态字段的值（放入常量池中的常量除外），或者调用一个静态方法的时候，对应类必须进行过初始化。

2）通过`java.lang.reflect`包的方法对类进行反射调用的时候，如果类没有进行过初始化，则要首先进行初始化。

3）当初始化一个类的时候，如果发现其父类没有进行过初始化，则首先触发父类初始化。

4）当虚拟机启动时，用户需要指定一个主类（包含`main()`方法的类），虚拟机会首先初始化这个类。

5）使用 jdk1.7 的动态语言支持时，如果一个`java.lang.invoke.MethodHandle`实例最后的解析结果`REF_getStatic`、`REF_putStatic`、`RE_invokeStatic`的方法句柄，并且这个方法句柄对应的类没有进行初始化，则需要先触发其初始化。

- **被动引用实例**

例子一：

通过子类引用父类的静态字段，对于父类属于“主动引用”的第一种情况，父类会被初始化；对于子类，没有符合“主动引用”的情况，故子类不会进行初始化。代码如下：

```java
//父类
class SuperClass {
	//静态变量value
	public static int value = 666;
	//静态块，父类初始化时会调用
	static{
		System.out.println("父类初始化！");
	}
}

//子类
class SubClass extends SuperClass{
	//静态块，子类初始化时会调用
	static{
		System.out.println("子类初始化！");
	}
}

//主类、测试类
public class InitTest {
	public static void main(String[] args){
		System.out.println(SubClass.value);
	}
}
```

输出结果：

```ASN.1
父类初始化！
666
```


例子二：

通过数组来引用类，不会触发类的初始化，因为是数组被new，但是类没有被new，所以没有触发任何“主动引用”情况，属于“被动引用”。代码如下：

```java
//父类
class SuperClass {
	//静态变量value
	public static int value = 666;
	//静态块，父类初始化时会调用
	static{
		System.out.println("父类初始化！");
	}
}

//主类、测试类
public class InitTest {
	public static void main(String[] args){
		SuperClass[] test = new SuperClass[10];
	}
}
```

没有任何结果输出。

例子三：

静态常量在编译阶段就会被存入调用类的常量池中，不会引用到定义常量的类，这是一个特例，不会触发类的初始化。

```java
//常量类
class ConstClass {
	static{
		System.out.println("常量类初始化！");
	}
	public static final String HELLOWORLD = "hello world!";

}

//主类、测试类
public class NotInit {
	public static void main(String[] args){
		System.out.println(ConstClass.HELLOWORLD);
	}
}
```

输出结果：

```ASN.1
hello world!
```

## 卸载

当一个类被判定为无用类时，才可以被卸载，需要同时满足如下条件：

1）类的所有实例都已被回收；

2）加载该类的`ClassLoader`已被回收；

3）该类对应的`java.lang.Class`对象没有在任何地方被引用。

对于满足上述 3 个条件的无用类，虚拟机可以对其进行回收，但并不是必然的，是否回收可通过`-Xnoclassgc`参数控制。在大量使用反射、动态代理等字节码框架、动态生成`JSP`以及`OSGi`这类频繁自定义`ClassLoader`的场景都需要虚拟机具备类卸载的功能，以保证永久代（特指`HotSpot`虚拟机）不会溢出。

