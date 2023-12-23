# 幕后字段
当 getter 和 setter 有一个为默认实现，或者在 getter 和 setter 中通过 filed 标识符引用幕后字段时，才会自动生成幕后字段。
```kotlin
class Person {
    var name = "wavever"
        set(value) = print(value)
}
```
或是
```kotlin
class Person {
    var name = "wavever"
        get() = "name is $field"
        set(value) = print(value)
}
```

才会生成幕后字段。


# 幕后属性
在 Koltin 中通过 private 修饰的属性是不会生成 getter 和 setter，其访问都是直接通过字段来访问，但如果此时需求是希望直接通过字段来访问，但同时又需要让外界可以获取到该属性值，那么就可以通过幕后属性来实现。

# 函数式接口
只有一个抽象方法的接口成为函数式接口或者单一抽象方法接口。函数式接口可以有多个非抽象成员，但只能有一个抽象成员。
```kotlin
fun interface SamTest{    
    fun fun1()    
    //只能定义一个抽象方法    
    //fun fun2()、    
    fun fun3(){        
        print("non abstract.")    
    }
 }
```
# SAM转换
对于函数式接口，可以通过 lambda 表达式实现 SAM 转换，从而使代码更简洁、更有可读性。使用 lambda 表达式可以替代手动创建实现函数式接口的类。通过 SAM (Single Abstract Method) 转换，Kotlin 可以将任何签名与接口单一方法签名匹配的 lambda 表达式转换为代码，动态实例化接口实现。
这是官方的说明，通俗解释就是只要lambda表达式的签名与函数式接口的方法的签名是匹配的，lambda表达式可以简化接口的创建。

# 顶层函数可见性
```kotlin

// 文件名：example.ktpackage foo

fun baz() { …… }
class Bar { …… }
```

* 如果你不使用任何可见性修饰符，默认为 public，这意味着你的声明将随处可见。
* 如果你声明为 private，它只会在声明它的文件内可见。
* 如果你声明为 internal，它会在相同模块内随处可见。
* protected 修饰符不适用于顶层声明。
```kotlin

// 文件名：example.ktpackage foo

private fun foo() { …… } // 在 example.kt 内可见

public var bar: Int = 5 // 该属性随处可见
    private set         // setter 只在 example.kt 内可见

internal val baz = 6    // 相同模块内可见
```

# 扩展
扩展是什么？
扩展是kotlin提供的功能，通过扩展可以在不实现继承和装饰模式的前提下，为类扩展新的方法，属性。

扩展是静态解析的，意味着扩展是在编译器确定其接收者类型的。扩展只对其接收者类型起作用。

如果类内定义相同签名的成员函数与扩展函数，如果调用时成员函数和扩展函数都适用，总是取成员函数。

# 扩展多个接收者的情况

如果在一个类内部声明另一个类的扩展函数，那么这个扩展函数同时具有2个接收者，一个是本类的`this`称为分发接收者，第二个是扩展类的`this`称为扩展接收者,默认情况下，kotlin会自动处理接收者，我们可以同时调用本类以及扩展类的方法，也可已使用`@`指定接收者类型。当扩展接收者与分发接收者函数签名发生冲突时，需要显式指定接收者类型
```kotlin

class Host(val hostname: String) {
    fun printHostname() { print(hostname) }
}

class Connection(val host: Host, val port: Int) {
    fun printPort() { print(port) }

    fun Host.printConnectionString() {
         printHostname()   // 调用 Host.printHostname()
        print(":")
         printPort()   // 调用 Connection.printPort()
    }

    fun connect() {
         /*……*/
         host.printConnectionString()   // 调用扩展函数
    }
}

fun main() {
    Connection(Host("kotl.in"), 443).connect()
    //Host("kotl.in").printConnectionString()  // 错误，该扩展函数在 Connection 外不可用
}
```
# 扩展的多态
扩展在声明为成员的情况下是可以在子类中覆写的，意味着扩展函数本身可以是动态解析的，对于分发接收者来说是动态解析的，对于扩展接收者来说，是静态解析的。
```kotlin

open class Base { }

class Derived : Base() { }

open class BaseCaller {
    open fun Base.printFunctionInfo() {
        println("Base extension function in BaseCaller")
    }

    open fun Derived.printFunctionInfo() {
        println("Derived extension function in BaseCaller")
    }

    fun call(b: Base) {
        b.printFunctionInfo()   // 调用扩展函数
    }
}

class DerivedCaller: BaseCaller() {
    override fun Base.printFunctionInfo() {
        println("Base extension function in DerivedCaller")
    }

    override fun Derived.printFunctionInfo() {
        println("Derived extension function in DerivedCaller")
    }
}

fun main() {
    BaseCaller().call(Base())   // “Base extension function in BaseCaller”
    DerivedCaller().call(Base())  // “Base extension function in DerivedCaller”——分发接收者虚拟解析
    DerivedCaller().call(Derived())  // “Base extension function in DerivedCaller”——扩展接收者静态解析
}
```
# 简单理解静态解析与动态解析
静态解析可以理解为在编译期间确定函数的接收者，而动态解析则是在运行期间确定函数的接收者。扩展是静态解析的，接收者是在定义扩展函数是指定的接收者。而对于分发接收者来说，扩展是动态解析的。通俗解释就是，子类可以覆写父类中的扩展函数。


# 比较
== 和 equal() 相同，=== 比较内存地址

# data
数据类和普通类的区别在于，数据类是final的，不能被继承。
数据类至少有一个属性是在构造方法中声明。
数据类在构造方法中声明后，会生成对应的getter/setter，componentN()，并且参与equals、hashcode、copy等方法的生成。
数据类可用于解构声明，解构声明依赖于componentN方法。
只有所有属性都声明在构造方法中，才会生成无参构造方法。

结合Gson使用的话，必须将所有属性都声明在构造方法中，以生成无参构造函数，Gson实例化对象会首先查找无参构造方法，否则使用Unsafe实例化对象，并不调用任何构造方法，导致默认值失效。
# @Jvmoverloads 
`@jvmoverloads`注解的作用就是:在有默认参数值的方法加上`@jvmoverloads`注解，则`Kotlin`就会暴露多个重载方法。可以减少写构造方法。

# 中缀函数

infix ： 中缀函数，主要使用在只有一个参数的成员函数或者扩展函数上，同时比一般函数具有可读性；
使用条件：1、函数必须只有一个参数；2、函数必须是扩展函数或者成员函数；3、必须使用infix修饰；

用扩展函数举个例子：

```kotlin
infix fun Int.add(num:Int):Int{
return this + num
}
```

调用的时候：
```kotlin
val sum = 1 add 1
```
# 泛型

关于泛型是指参数化类型，是以类型作为参数，区别于以对象作为参数的编程方式。Java泛型并不“完善”只在编译期为类型检查提供帮助，可能是为了兼容旧版本，编译后的类型参数全部被抹除。假如我们定义了`List<String> strList`那么在编译后的字节码层面定义变成了`List strList`。这就是类型擦除，由于类型擦除，泛型类型支持的功能有限，我们无法利用泛型类型实例化变量，也无法对泛型类型进行反射。

Kotlin的泛型也存在同样的问题。

但是Kotlin的泛型对Java泛型有诸多改进，如协变与逆协变。
我们知道假如有2个类：
```kotlin
class Base
class Derived:Base
//定义一个方法
fun print(base:Base) { }
//下面2个调用都是可以编译通过的
print(Base())
print(Derived())

//假如Base作为泛型参数
fun print(bases:List<Base>) { }
//这句代码可以编译通过
print(ArrayList<Base>())
//这句会报错
print(ArrayList<Derived>())

```
原因是：Derived是Base的子类，但List<Derived>却不是List<Base>的子类。为了使泛型也具有这样的特性，Java发明了extends关键字，用来解决以上问题。
```kotlin
//假如Base作为泛型参数
fun print(bases:List<? extends Base>) { }
//这句代码可以编译通过
print(ArrayList<Base>())
//这次编译通过
print(ArrayList<Derived>())
```
通过extends关键字，Base的继承关系映射到List<Base>,这个过程称为协变。网上还流传者上界与下界的说法，我认为这种说法太过抽象，无法看出本质。至于为什么用extends修饰的泛型类型只能get?现在解释起来就比较容易，`List<? extends Base>`类型参数的类型可以接受任何Base子类类型的`List`所以，编译其无法确定`List<? extends Base>`参数的泛型类型具体是什么类型，所以无法set，同时还规定了类型上界为Base。
用同样的原理解释super关键字，与extends相反`List<? super Derived>`表示类型参数的类型可以是Derived以及它的父类型，比如Base所以`List<Derived>`和`List<Base>`都可以作为参数传递到方法中，但是由于无法确定传入类型参数是Derived的那个父类，所以使用get无法得到具体的类型，只能得到Obect类型的对象。由于规定了类型的下界，所以可以set Derived类型以及Derived的子类型。这称为逆协变。

在Kotlin中out对应extends，in对应super。

# 声明处型变
形变可以定义在泛型声明的地方。
# 使用处形变
在Kotlin中out对应extends，in对应super。

