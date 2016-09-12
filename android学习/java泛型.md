> java泛型解决容器，不确定类型问题，多个返回值，避免类型转换。

# 类泛型
类泛型定义的时候需要在类型后增加尖括号，和参数类型，如下所示:

```javascript
public class Holder<T> {
    private T a;

    public Holder(T a) {
        this.a = a;
    }

    public void set(T a) {
        this.a = a;
    }

    public T get() {
        return a;
    }
}
调用:
 Holder<String> h1 = new Holder<>(new String("aaa"));
 String test = h1.get();
 System.out.println(test);
```
这样Holder类既可以接收任意类型的数据，并保存在变量a中。
# 接口泛型
```javascript
1. 定义接口
public interface Generator<T> {
    public T next();
}
2. 实现接口
public class FruitGenerator implements Generator<String> {

    private String[] fruits = new String[]{"Apple", "Banana", "Pear"};

    @Override
    public String next() {
        Random rand = new Random();
        return fruits[rand.nextInt(3)];
    }
}
3. 调用
public class Main {

    public static void main(String[] args) {
        FruitGenerator generator = new FruitGenerator();
        System.out.println(generator.next());
        System.out.println(generator.next());
        System.out.println(generator.next());
        System.out.println(generator.next());
    }
}
```
# 数组
假设有如下的一个类:<br>

```javascript
class Generic<T> {
    
}
```
定义泛型数组有两种方法：<br>
* 使用ArrayList
 
```javascript
public class ListOfGenerics<T> {
    private List<T> array = new ArrayList<T>();
    public void add(T item) { array.add(item); }
    public T get(int index) { return array.get(index); }
}
```

* 使用泛型，方法如下

```javascript
public class GenericArrayWithTypeToken<T> {
    private T[] array;
    @SuppressWarnings("unchecked")
    public GenericArrayWithTypeToken(Class<T> type, int sz) {
        array = (T[])Array.newInstance(type, sz);
    }
    public void put(int index, T item) {
        array[index] = item;
    }
    public T get(int index) { return array[index]; }
    // Expose the underlying representation:
    public T[] rep() { return array; }
    public static void main(String[] args) {
        GenericArrayWithTypeToken<Integer> gai =
        new GenericArrayWithTypeToken<Integer>(
        Integer.class, 10);
        // This now works:
        Integer[] ia = gai.rep();
    }
}
在构造器中传入了 Class<T> 对象，通过 Array.newInstance(type, sz) 创建一个数组，这个方法会用参数中的 Class 对象作为数组元素的组件类型。这样创建出的数组的元素类型便不再是 Object，而是 T。这个方法返回 Object 对象，需要把它转型为数组。不过其他操作都不需要转型了，包括 rep() 方法，因为数组的实际类型与 T[] 是一致的。这是比较推荐的创建泛型数组的方法。
```
Java 中不允许直接创建泛型数组。

# 泛型和子类型
假设Apple是Fruit的子类。下面代码是否正确:<br>

```javascript
List<Apple> apples = new ArrayList<Apple>();
List<Fruit> fruits = apples;
```
第一行代码肯定是对的，但是第二行代码不正确。这样考虑，假定第二行代码没有问题，是否可以使用语句fruits.add(new Strawberry())，其中Strawberry为Fruit的子类，这样就可以在fruits中加入草莓，但是这样的话，fruits真正只想的对象为apples，其中只能添加Apple对象的。因此第二行代码不正确。<br>
通常来说，如果Foo是Bar的子类，G是一种带泛型的类型，则G<Foo>不是G<Bar>的子类型。
# 类型擦除
Java中的泛型基本上都是在编译器这个层次来实现的。在生成的Java字节码中是不包含泛型中的类型信息的。使用泛型的时候加上的类型参数，会在编译器在编译的时候去掉。这个过程就称为类型擦除。对每个泛型类只生成唯一的一份目标代码；该泛型类的所有实例都映射到这份目标代码上，在需要的时候执行类型检查和类型转换<br>
在泛型代码内部，无法获得任何有关泛型参数类型的信息<br>
```javascript
public class ErasedTypeEquivalence {
    public static void main(String[] args) {
        Class c1 = new ArrayList<String>().getClass();
        Class c2 = new ArrayList<Integer>().getClass();
        System.out.println(c1 == c2);
    }
}
输出值为：true
```
## 产生的问题
丢失了参数的类型。不能执行参数的方法。
### 重载
```javascript
public class GenericTypes {  

    public static void method(List<String> list) {  
        System.out.println("invoke method(List<String> list)");  
    }  

    public static void method(List<Integer> list) {  
        System.out.println("invoke method(List<Integer> list)");  
    }  
}
上面的两个方法是相同的。
```
### catch异常
如果我们自定义了一个泛型异常类GenericException，那么，不要尝试用多个catch取匹配不同的异常类型，例如你想要分别捕获GenericException、GenericException，这也是有问题的。
### 当泛型内包含静态变量
```javascript
ublic class StaticTest{
    public static void main(String[] args){
        GT<Integer> gti = new GT<Integer>();
        gti.var=1;
        GT<String> gts = new GT<String>();
        gts.var=2;
        System.out.println(gti.var);
    }
}
class GT<T>{
    public static int var=0;
    public void nothing(T x){}
}
```
此时var的值为2，因为经过类型擦除，所有的泛型类实例都关联到同一份字节码上，泛型类的所有静态变量是共享的。
### instanceOf
不能对确切的泛型类型使用instanceOf操作。
### 泛型类型是被所有调用共享
所有泛型类的实例都共享同一个运行时类。
## 擦除的补偿
类型擦除导致了泛型丧失了一些功能，任何运行期需要知道确切类型的代码都无法工作。以下代码无法正常工作:<br>
```javascript
public class Erased<T> {
    private final int SIZE = 100;
    public static void f(Object arg) {
    if(arg instanceof T) {} // Error
    T var = new T(); // Error
    T[] array = new T[SIZE]; // Error
    T[] array = (T)new Object[SIZE]; // Unchecked warning
    }
}
```
通过 new T() 创建对象是不行的，一是由于类型擦除，二是由于编译器不知道 T 是否有默认的构造器。一种解决的办法是传递一个工厂对象并且通过它创建新的实例。<br>
```javascript
interface FactoryI<T> {
    T create();
}
class Foo2<T> {
    private T x;
    public <F extends FactoryI<T>> Foo2(F factory) {
    x = factory.create();
    }
    // ...
}
class IntegerFactory implements FactoryI<Integer> {
    public Integer create() {
    return new Integer(0);
    }
}
class Widget {
    public static class Factory implements FactoryI<Widget> {
        public Widget create() {
            return new Widget();
        }
    }
}
public class FactoryConstraint {
    public static void main(String[] args) {
        new Foo2<Integer>(new IntegerFactory());
        new Foo2<Widget>(new Widget.Factory());
    }
}
```
另外一种方法是利用末班设计模式：<br>

```javascript
abstract class GenericWithCreate<T> {
    final T element;
    GenericWithCreate() { element = create(); }
    abstract T create();
}
class X {}
class Creator extends GenericWithCreate<X> {
    X create() { return new X(); }
    void f() {
    System.out.println(element.getClass().getSimpleName());
    }
}
public class CreatorGeneric {
    public static void main(String[] args) {
        Creator c = new Creator();
        c.f();
    }
}
```
具体类型的创建放到了子类继承父类时，在 create 方法中创建实际的类型并返回。
# 边界
## 协变
首先看如下的代码:<br>

```javscript
class Fruit {}
class Apple extends Fruit {}
class Jonathan extends Apple {}
class Orange extends Fruit {}

public class CovariantArrays {
    public static void main(String[] args) {       
        Fruit[] fruit = new Apple[10];
        fruit[0] = new Apple(); // OK
        fruit[1] = new Jonathan(); // OK
        // Runtime type is Apple[], not Fruit[] or Orange[]:
        try {
            // Compiler allows you to add Fruit:
            fruit[0] = new Fruit(); // ArrayStoreException
        } catch(Exception e) { System.out.println(e); }
        try {
            // Compiler allows you to add Oranges:
            fruit[0] = new Orange(); // ArrayStoreException
        } catch(Exception e) { System.out.println(e); }
        }
} /* Output:
java.lang.ArrayStoreException: Fruit
java.lang.ArrayStoreException: Orange
```
main 方法中的第一行，创建了一个 Apple 数组并把它赋给 Fruit 数组的引用。这是有意义的，Apple 是 Fruit 的子类，一个 Apple 对象也是一种 Fruit 对象，所以一个 Apple 数组也是一种 Fruit 的数组。这称作数组的协变。尽管 Apple[] 可以 “向上转型” 为 Fruit[]，但数组元素的实际类型还是 Apple，我们只能向数组中放入 Apple或者 Apple 的子类。在上面的代码中，向数组中放入了 Fruit 对象和 Orange 对象。对于编译器来说，这是可以通过编译的，但是在运行时期，JVM 能够知道数组的实际类型是 Apple[]，所以当其它对象加入数组的时候就会抛出异常。<br>
泛型设计的目的之一是要使这种运行时期的错误在编译期就能发现，看看用泛型容器类来代替数组会发生什么：

```javscript
// Compile Error: incompatible types:
ArrayList<Fruit> flist = new ArrayList<Apple>();
```
上面的代码根本就无法编译。当涉及到泛型时， 尽管 Apple 是 Fruit 的子类型，但是 ArrayList<Apple> 不是 ArrayList<Fruit> 的子类型，泛型不支持协变。
## 上边界
利用 <? extends Fruit> 形式的通配符，可以实现泛型的向上转型：<br>

```
public class GenericsAndCovariance {
    public static void main(String[] args) {
        // Wildcards allow covariance:
        List<? extends Fruit> flist = new ArrayList<Apple>();
        // Compile Error: can’t add any type of object:
        // flist.add(new Apple());
        // flist.add(new Fruit());
        // flist.add(new Object());
        flist.add(null); // Legal but uninteresting
        // We know that it returns at least Fruit:
        Fruit f = flist.get(0);
    }
}
```
通配符 List<? extends Fruit> 表示某种特定类型 ( Fruit 或者其子类 ) 的 List，但是并不关心这个实际的类型到底是什么，反正是 Fruit 的子类型，Fruit 是它的上边界。那么对这样的一个 List 我们能做什么呢？其实如果我们不知道这个 List 到底持有什么类型，怎么可能安全的添加一个对象呢？在上面的代码中，向 flist 中添加任何对象，无论是 Apple 还是 Orange 甚至是 Fruit 对象，编译器都不允许，唯一可以添加的是 null。所以如果做了泛型的向上转型 (List<? extends Fruit> flist = new ArrayList<Apple>())，那么我们也就失去了向这个 List 添加任何对象的能力，即使是 Object 也不行。<br>
我们只知道flist中的元素是Fruit的类型的子类型，具体是什么类型我们并不清楚，所有不能加入任何类型的对象。<br>
另一方面，如果调用某个返回 Fruit 的方法，这是安全的。因为我们知道，在这个 List 中，不管它实际的类型到底是什么，但肯定能转型为 Fruit，所以编译器允许返回 Fruit。
## 下边界
通配符的另一个方向是　“超类型的通配符“: ? super T，T 是类型参数的下界。使用这种形式的通配符，我们就可以 ”传递对象” 了。还是用例子解释：<br>

```javascript
public class SuperTypeWildcards {
    static void writeTo(List<? super Apple> apples) {
        apples.add(new Apple());
        apples.add(new Jonathan());
        // apples.add(new Fruit()); // Error
    }
}
```
writeTo 方法的参数 apples 的类型是 List<? super Apple>，它表示某种类型的 List，这个类型是 Apple 的基类型。也就是说，我们不知道实际类型是什么，但是这个类型肯定是 Apple 的父类型。因此，我们可以知道向这个 List 添加一个 Apple 或者其子类型的对象是安全的，这些对象都可以向上转型为 Apple。但是我们不知道加入 Fruit 对象是否安全，因为那样会使得这个 List 添加跟 Apple 无关的类型。可以写入子类包括自己的对象。
## 无边界通配符
它的使用形式是一个单独的问号：List<?>，也就是没有任何限定。不做任何限制，跟不用类型参数的 List 有什么区别呢？

List<?> list 表示 list 是持有某种特定类型的 List，但是不知道具体是哪种类型。那么我们可以向其中添加对象吗？当然不可以，因为并不知道实际是哪种类型，所以不能添加任何类型，这是不安全的。而单独的 List list ，也就是没有传入泛型参数，表示这个 list 持有的元素的类型是 Object，因此可以添加任何类型的对象，只不过编译器会有警告信息。
## 边界通配符总结
* 如果想从一个数据类型中获取数据，使用 ? extends 通配符
* 如果想把对象写入一个数据结构里，使用 ? super 通配符
* 如果既想存，又想取，就别使用通配符