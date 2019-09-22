# Java语法糖

语法糖（Syntactic Sugar），也称糖衣语法，是由英国计算机学家 Peter.J.Landin 发明的一个术语，指在计算机语言中添加的某种语法，这种语法对语言的功能并没有影响，但是更方便程序员使用。简而言之，语法糖让程序更加简洁，有更高的可读性。

> 有意思的是，在编程领域，除了语法糖，还有语法盐、语法糖精、语法海洛因的说法，篇幅有限这里不做扩展了。

我们所熟知的编程语言中几乎都有语法糖。很多人说Java是一个“低糖语言”，其实从Java 7开始Java语言层面上一直在添加各种糖，主要是在“Project Coin”项目下研发。尽管现在Java有人还是认为现在的Java是低糖，未来还会持续向着“高糖”的方向发展。

#### 解语法糖

前面提到过，语法糖的存在主要是方便开发人员使用。但其实，Java虚拟机并不支持这些语法糖。这些语法糖在编译阶段就会被还原成简单的基础语法结构，这个过程就是解语法糖。

说到编译，大家肯定都知道，Java语言中，javac命令可以将后缀名为.java的源文件编译为后缀名为.class的可以运行于Java虚拟机的字节码。

如果你去看com.sun.tools.javac.main.JavaCompiler的源码，你会发现在compile()中有一个步骤就是调用desugar()，这个方法就是负责解语法糖的实现的。

Java 中最常用的语法糖主要有泛型、变长参数、条件编译、自动拆装箱、内部类等。今天主要来分析下这些语法糖背后的原理。一步一步剥去糖衣，看看其本质。

#### 糖块一、 switch 支持 String 与枚举

前面提到过，从Java 7 开始，Java语言中的语法糖在逐渐丰富，其中一个比较重要的就是Java 7中switch开始支持String。Java中的swith自身原本就支持基本类型。比如int、char等。对于int类型，直接进行数值的比较，对于char类型则是比较其ascii码。所以，对于编译器来说，switch中其实只能使用整型，任何类型的比较都要转换成整型。比如byte。short，char(ascii码是整型)以及int。

```
public class SwitchDemoString {
    public static void main(String[] args) {
        String str = "world";
        switch (str) {
        case "hello":
            System.out.println("hello");
            break;
        case "world":
            System.out.println("world");
            break;
        default:
            break;
        }
    }
}
```

反编译后内容如下：

```
public class SwitchDemoString
{
    public SwitchDemoString()
    {
    }
    public static void main(String args[])
    {
        String str = "world";
        String s;
        switch((s = str).hashCode())
        {
        default:
            break;
        case 99162322:
            if(s.equals("hello"))
                System.out.println("hello");
            break;
        case 113318802:
            if(s.equals("world"))
                System.out.println("world");
            break;
        }
    }
}
```

看到这个代码，才知道原来字符串的switch是通过equals()和hashCode()方法来实现的。还好hashCode()方法返回的是int，而不是long。

> 仔细看下可以发现，进行switch的实际是哈希值，然后通过使用equals方法比较进行安全检查，这个检查是必要的，因为哈希可能会发生碰撞。因此它的性能是不如使用枚举进行switch或者使用纯整数常量，但这也不是很差。

#### 糖块二、 泛型

我们都知道，很多语言都是支持泛型的，但是很多人不知道的是，不同的编译器对于泛型的处理方式是不同的。

通常情况下，一个编译器处理泛型有两种方式：Code specialization和Code sharing。

C++和C#是使用Code specialization的处理机制，而Java使用的是Code sharing的机制。

> Code sharing方式为每个泛型类型创建唯一的字节码表示，并且将该泛型类型的实例都映射到这个唯一的字节码表示上。将多种泛型类形实例映射到唯一的字节码表示是通过类型擦除（type erasue）实现的。

也就是说，对于Java虚拟机来说，他根本不认识Map<String, String> map这样的语法。需要在编译阶段通过类型擦除的方式进行解语法糖。
```
Map<String, String> map = new HashMap<>();
map.put("key1", "value1");
map.put("key2", "value2");
map.put("key3", "value3");
```

解语法糖之后会变成：

```
Map map = new HashMap();
map.put("key1", "value1");
map.put("key2", "value2");
map.put("key3", "value3");
```

```
public static <A extends Comparable<A>> A max(Collection<A> xs) {
    Iterator<A> xi = xs.iterator();
    A w = xi.next();
    while (xi.hasNext()) {
        A x = xi.next();
        if (w.compareTo(x) < 0)
            w = x;
    }
    return w;
}
```

类型擦除后会变成：

```
public static Comparable max(Collection xs){
    Iterator xi = xs.iterator();
    Comparable w = (Comparable)xi.next();
    while(xi.hasNext())
    {
        Comparable x = (Comparable)xi.next();
        if(w.compareTo(x) < 0)
            w = x;
    }
    return w;
}
```

虚拟机中没有泛型，只有普通类和普通方法，所有泛型类的类型参数在编译时都会被擦除，泛型类并没有自己独有的Class类对象。比如并不存在List<String>.class或是List<Integer>.class，而只有List.class。

#### 糖块三、 自动装箱与拆箱

自动装箱就是Java自动将原始类型值转换成对应的对象，比如将int的变量转换成Integer对象，这个过程叫做装箱，反之将Integer对象转换成int类型值，这个过程叫做拆箱。

原始类型byte, short, char, int, long, float, double 和 boolean 对应的封装类为Byte, Short, Character, Integer, Long, Float, Double, Boolean。

先来看个自动装箱的代码：

```
public static void main(String[] args) {
    int i = 10;
    Integer n = i;
}
```

反编译后代码如下:

```
public static void main(String args[])
{
    int i = 10;
    Integer n = Integer.valueOf(i);
}
```

再来看个自动拆箱的代码：

```
public static void main(String[] args) {

    Integer i = 10;
    int n = i;
}
```

反编译后代码如下：

```
public static void main(String args[])
{
    Integer i = Integer.valueOf(10);
    int n = i.intValue();
}
```

从反编译得到内容可以看出，在装箱的时候自动调用的是Integer的valueOf(int)方法。而在拆箱的时候自动调用的是Integer的intValue方法。

#### 糖块四 、 方法变长参数

可变参数(variable arguments)是在Java 1.5中引入的一个特性。它允许一个方法把任意数量的值作为参数。

看下以下可变参数代码，其中print方法接收可变参数：

```
public static void print(String... strs) {
        for (int i = 0; i < strs.length; i++) {
            System.out.println(strs[i]);
        }
    }
public static void main(String[] args) {
        print("Method", "Variable", "Parameter");
    }
```

反编译后代码：

```
public static transient void print(String strs[])
    {
        for(int i = 0; i < strs.length; i++)
            System.out.println(strs[i]);

    }

public static void main(String args[])
    {
        print(new String[] {
            "Method", "Variable", "Parameter"
        });
    }
```

从反编译后代码可以看出，可变参数在被使用的时候，首先会创建一个数组，数组的长度就是调用该方法时传递的实参的个数，然后再把参数值全部放到这个数组当中，然后再把这个数组作为参数传递到被调用的方法中。

#### 糖块五 、 枚举

```
public enum Season {
    SPRING,SUMMER;
}
```

然后我们使用反编译，看看这段代码到底是怎么实现的，反编译后代码内容如下：

```
public final class Season extends Enum
{
    public static Season[] values()
    {
        return (Season[])$VALUES.clone();
    }
    public static Season valueOf(String name)
    {
        return (Season)Enum.valueOf(sugar/Season, name);
    }

    private Season(String s, int i)
    {
        super(s, i);
    }

    public static final Season SPRING;
    public static final Season SUMMER;
    private static final Season $VALUES[];

    static 
    {
        SPRING = new Season("SPRING", 0);
        SUMMER = new Season("SUMMER", 1);
        $VALUES = (new Season[] {
            SPRING, SUMMER
        });
    }
}
```

通过反编译后代码我们可以看到，public final class T extends Enum，说明，该类是继承了Enum类的，同时final关键字告诉我们，这个类也是不能被继承的。

当我们使用enmu来定义一个枚举类型的时候，编译器会自动帮我们创建一个final类型的类继承Enum类，所以枚举类型不能被继承。

#### 糖块六 、 内部类

内部类又称为嵌套类，可以把内部类理解为外部类的一个普通成员。

内部类之所以也是语法糖，是因为它仅仅是一个编译时的概念。

```
public class OutterClass {
    private String userName;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public static void main(String[] args) {

    }

    class InnerClass{
        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
}
```

```
class OutterClass$InnerClass
{

    public String getName()
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }

    private String name;
    final OutterClass this$0;

    OutterClass$InnerClass()
    {
        this.this$0 = OutterClass.this;
        super();
    }
}
```

#### 糖块七 、条件编译

—般情况下，程序中的每一行代码都要参加编译。但有时候出于对程序代码优化的考虑，希望只对其中一部分内容进行编译，此时就需要在程序中加上条件，让编译器只对满足条件的代码进行编译，将不满足条件的代码舍弃，这就是条件编译。

```
public class ConditionalCompilation {
    public static void main(String[] args) {
        final boolean DEBUG = true;
        if(DEBUG) {
            System.out.println("Hello, DEBUG!");
        }

        final boolean ONLINE = false;
        if(ONLINE){
            System.out.println("Hello, ONLINE!");
        }
    }
}
```

反编译后代码如下：

```
public class ConditionalCompilation
{

    public ConditionalCompilation()
    {
    }

    public static void main(String args[])
    {
        boolean DEBUG = true;
        System.out.println("Hello, DEBUG!");
        boolean ONLINE = false;
    }
}
```

首先，我们发现，在反编译后的代码中没有System.out.println("Hello, ONLINE!");，这其实就是条件编译。

当if(ONLINE)为false的时候，编译器就没有对其内的代码进行编译。

所以，Java语法的条件编译，是通过判断条件为常量的if语句实现的。根据if判断条件的真假，编译器直接把分支为false的代码块消除。通过该方式实现的条件编译，必须在方法体内实现，而无法在正整个Java类的结构或者类的属性上进行条件编译。

#### 糖块八 、 断言

assert关键字是从JAVA SE 1.4 引入的，为了避免和老版本的Java代码中使用了assert关键字导致错误，Java在执行的时候默认是不启动断言检查的（这个时候，所有的断言语句都将忽略！）。

如果要开启断言检查，则需要用开关-enableassertions或-ea来开启。

```
public class AssertTest {
    public static void main(String args[]) {
        int a = 1;
        int b = 1;
        System.out.println("assert start");
        assert a == b;
        System.out.println("程序正常");
        assert a != b : "程序异常";
        System.out.println("assert end");
    }
}
```

反编译后代码如下：

```
public class AssertTest
{
    public AssertTest()
    {
    }
    public static void main(String args[])
    {
        int a = 1;
        int b = 1;
        System.out.println("assert start");
        if(!$assertionsDisabled && a != b)
            throw new AssertionError();
        System.out.println("\u7A0B\u5E8F\u6B63\u5E38");
        if(!$assertionsDisabled && a == b)
        {
            throw new AssertionError("\u7A0B\u5E8F\u5F02\u5E38");
        } else
        {
            System.out.println("assert end");
            return;
        }
    }
    static final boolean $assertionsDisabled = !sugar/AssertTest.desiredAssertionStatus();
}
```

其实断言的底层实现就是if语言，如果断言结果为true，则什么都不做，程序继续执行，如果断言结果为false，则程序抛出AssertError来打断程序的执行。

#### 糖块九 、 数值字面量

在java 7中，数值字面量，不管是整数还是浮点数，都允许在数字之间插入任意多个下划线。这些下划线不会对字面量的数值产生影响，目的就是方便阅读。

```
public class NumberLiteral {
    public static void main(String... args) {
        int i = 10_0_0__0;
        float f = 3.14_15__9f;
        System.out.println(i);
        System.out.println(f);
    }
}
```

反编译后：

```
public class NumberLiteral
{
    public NumberLiteral()
    {
    }
    public static transient void main(String args[])
    {
        int i = 10000;
        float f = 3.14159F;
        System.out.println(i);
        System.out.println(f);
    }
}
```

反编译后就是把 _ 删除了。也就是说编译器并不认识在数字字面量中的_，需要在编译阶段把他去掉。

#### 糖块十 、 for-each

增强for循环（for-each）相信大家都不陌生，日常开发经常会用到的，它会比for循环要少写很多代码，那么这个语法糖背后是如何实现的呢？

```
    public static void main(String... args) {
        String[] strs = {"str1", "str2", "str3"};
        for (String s : strs) {
            System.out.println(s);
        }
        List<String> strList = Arrays.asList("test1", "test1", "test1");
        for (String s : strList) {
            System.out.println(s);
        }
    }
```

反编译后代码如下：

```
    public static transient void main(String args[])
    {
        String strs[] = {
            "str1", "str2", "str3"
        };
        String args1[] = strs;
        int i = args1.length;
        for(int j = 0; j < i; j++)
        {
            String s = args1[j];
            System.out.println(s);
        }

        List strList = Arrays.asList(new String[] {
            "test1", "test1", "test1"
        });
        String s;
        for(Iterator iterator = strList.iterator(); iterator.hasNext(); System.out.println(s))
            s = (String)iterator.next();

    }
```

代码很简单，for-each的实现原理其实就是使用了普通的for循环和迭代器。

#### 糖块十一 、 try-with-resource

对于文件操作IO流、数据库连接等开销非常昂贵的资源，用完之后必须及时通过close方法将其关闭，否则资源会一直处于打开状态，可能会导致内存泄露等问题。

关闭资源的常用方式就是在finally块里是释放，即调用close方法。比如，我们经常会写这样的代码：

```
        BufferedReader br = null;
        try {
            String line;
            br = new BufferedReader(new FileReader("d:/test.txt"));
            while ((line = br.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            // handle exception
        } finally {
            try {
                if (br != null) {
                    br.close();
                }
            } catch (IOException ex) {
                // handle exception
            }
        }
```

从Java 7开始，jdk提供了一种更好的方式关闭资源，使用try-with-resources语句，改写一下上面的代码，效果如下：

```
        try (BufferedReader br = new BufferedReader(new FileReader("d:/test.txt"))) {
            String line;
            while ((line = br.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
```

这种新的语法糖看上去好像优雅很多，反编译看下它的背后原理：

```
        BufferedReader br;
        Throwable throwable;
        br = new BufferedReader(new FileReader("d:/test.txt"));
        throwable = null;
        String line;
        try
        {
            while((line = br.readLine()) != null) 
                System.out.println(line);
        }
        catch(Throwable throwable2)
        {
            throwable = throwable2;
            throw throwable2;
        }
        if(br != null)
            if(throwable != null)
                try
                {
                    br.close();
                }
                catch(Throwable throwable1)
                {
                    throwable.addSuppressed(throwable1);
                }
            else
                br.close();
        break MISSING_BLOCK_LABEL_117;
        Exception exception;
        exception;
        if(br != null)
            if(throwable != null)
                try
                {
                    br.close();
                }
                catch(Throwable throwable3)
                {
                    throwable.addSuppressed(throwable3);
                }
            else
                br.close();
        throw exception;
        IOException e;
        e;
        e.printStackTrace();
```

其实背后的原理也很简单，那些我们没有做的关闭资源的操作，编译器都帮我们做了。

所以，再次印证了，语法糖的作用就是方便程序员的使用，但最终还是要转成编译器认识的语言。

#### 糖块十二、Lambda表达式

关于lambda表达式，有人可能会有质疑，因为网上有人说他并不是语法糖。但其实它是一个语法糖。实现方式是依赖了几个JVM底层提供的lambda相关api。

先来看一个简单的lambda表达式。遍历一个list：

```
    public static void main(String[] args) {
        List<String> strList = Arrays.asList("test1", "test2", "test3");
        strList.forEach(e -> System.out.println(e));
    }
```

反编译后代码如下: java -jar .\cfr-0.146.jar .\LambdaTest.class --decodelambdas false

```
    public static void main(String[] arrstring) {
        List<String> list = Arrays.asList("test1", "test2", "test3");
        list.forEach((Consumer<String>)LambdaMetafactory.metafactory(null, null, null, (Ljava/lang/Object;)V, lambda$main$0(java.lang.String ), (Ljava/lang/String;)V)());
    }

    private static /* synthetic */ void lambda$main$0(String string) {
        System.out.println(string);
    }
```

可以看到，在forEach方法中，其实是调用了java.lang.invoke.LambdaMetafactory#metafactory方法，该方法的第5个参数implMethod指定了方法实现。可以看到这里其实是调用了一个lambda$main$0方法进行了输出。

再来看一个稍微复杂一点的，先对List进行过滤，然后再输出：

```
    public static void main(String[] args) {
        List<String> strList = Arrays.asList("test1", "test2", "test3");
        strList.forEach(e -> System.out.println(e));
        List<String> newList = strList.stream().filter(str -> str.contains("2")).collect(Collectors.toList());
        newList.forEach(e -> System.out.println(e));
    }
```

反编译后代码如下：

```
    public static void main(String[] arrstring) {
        List<String> list = Arrays.asList("test1", "test2", "test3");
        list.forEach((Consumer<String>)LambdaMetafactory.metafactory(null, null, null, (Ljava/lang/Object;)V, lambda$main$0(java.lang.String ), (Ljava/lang/String;)V)());
        List<Object> list2 = list.stream().filter((Predicate<String>)LambdaMetafactory.metafactory(null, null, null, (Ljava/lang/Object;)Z, lambda$main$1(java.lang.String ), (Ljava/lang/String;)Z)()).collect(Collectors.toList());
        list2.forEach((Consumer<Object>)LambdaMetafactory.metafactory(null, null, null, (Ljava/lang/Object;)V, lambda$main$2(java.lang.Object ), (Ljava/lang/Object;)V)());
    }

    private static /* synthetic */ void lambda$main$2(Object object) {
        System.out.println(object);
    }

    private static /* synthetic */ boolean lambda$main$1(String string) {
        return string.contains("2");
    }

    private static /* synthetic */ void lambda$main$0(String string) {
        System.out.println(string);
    }
```

所以，lambda表达式的实现其实是依赖了一些底层的api，在编译阶段，编译器会把lambda表达式进行解糖，转换成调用内部api的方式。

### **总结**

前面介绍了12种Java中常用的语法糖。所谓语法糖就是提供给开发人员便于开发的一种语法而已。

但是这种语法只有开发人员认识。要想被执行，需要进行解糖，即转成JVM认识的语法。

当我们把语法糖解糖之后，你就会发现其实我们日常使用的这些方便的语法，其实都是一些其他更简单的语法构成的。

有了这些语法糖，我们在日常开发的时候可以大大提升效率，但是同时也要避免过渡使用。使用之前最好了解下原理，避免掉坑。