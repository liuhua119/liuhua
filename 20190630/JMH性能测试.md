# JMH性能测试

​        JMH，即Java Microbenchmark Harness，是专门用于代码微基准测试的工具套件。何谓Micro Benchmark呢？简单的来说就是基于方法层面的基准测试，精度可以达到微秒级。当你定位到热点方法，希望进一步优化方法性能的时候，就可以使用JMH对优化的结果进行量化的分析。

​       JMH比较典型的应用场景有：

- 想准确的知道某个方法需要执行多长时间，以及执行时间和输入之间的相关性；
- 对比接口不同实现在给定条件下的吞吐量，找到最优实现；
- 查看多少百分比的请求在多长时间内完成。

看一个示例：(需要引入相关的依赖)

```
		<dependency>
			<groupId>org.openjdk.jmh</groupId>
			<artifactId>jmh-core</artifactId>
			<version>1.19</version>
		</dependency>
		<dependency>
			<groupId>org.openjdk.jmh</groupId>
			<artifactId>jmh-generator-annprocess</artifactId>
			<version>1.19</version>
			<scope>provided</scope>
		</dependency>
		
		<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-shade-plugin</artifactId>
				<version>2.0</version>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>shade</goal>
						</goals>
						<configuration>
							<finalName>microbenchmarks</finalName>
							<transformers>
								<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
									<mainClass>org.openjdk.jmh.Main</mainClass>
								</transformer>
							</transformers>
						</configuration>
					</execution>
				</executions>
			</plugin>
```

```
@BenchmarkMode(Mode.AverageTime) // 测试方法平均执行时间
@OutputTimeUnit(TimeUnit.MICROSECONDS) // 输出结果的时间粒度为微秒
@State(Scope.Thread) // 每个测试线程一个实例
public class FirstBenchMark {
    private static Logger log = LoggerFactory.getLogger(FirstBenchMark.class);
    @Benchmark
    public String stringConcat() {
        String a = "a";
        String b = "b";
        String c = "c";
        String s = a + b + c;
        log.debug(s);
        return s;
    }
    public static void main(String[] args) throws RunnerException {
        // 使用一个单独进程执行测试，执行5遍warmup，然后执行5遍测试
        Options opt = new OptionsBuilder().include(FirstBenchMark.class.getSimpleName()).forks(1).warmupIterations(5)
                .measurementIterations(5).build();
        new Runner(opt).run();
    }
}
```

1. Mode
Mode 表示 JMH 进行 Benchmark 时所使用的模式。通常是测量的维度不同，或是测量的方式不同。目前 JMH 共有四种模式：
> 1. Throughput: 整体吞吐量，例如“1秒内可以执行多少次调用”。
> 2. AverageTime: 调用的平均时间，例如“每次调用平均耗时xxx毫秒”。
> 3. SampleTime: 随机取样，最后输出取样结果的分布，例如“99%的调用在xxx毫秒以内，99.99%的调用在xxx毫秒以内”
> 4. SingleShotTime: 以上模式都是默认一次 iteration 是 1s，唯有 SingleShotTime 是只运行一次。往往同时把 warmup 次数设为0，用于测试冷启动时的性能。

2. Iteration
   Iteration 是 JMH 进行测试的最小单位。在大部分模式下，一次 iteration 代表的是一秒，JMH 会在这一秒内不断调用需要 benchmark 的方法，然后根据模式对其采样，计算吞吐量，计算平均执行时间等。

3. Warmup
   Warmup 是指在实际进行 benchmark 前先进行预热的行为。为什么需要预热？因为 JVM 的 JIT 机制的存在，如果某个函数被调用多次之后，JVM 会尝试将其编译成为机器码从而提高执行速度。为了让 benchmark 的结果更加接近真实情况就需要进行预热。

###  注解说明

1. @BenchmarkMode
对应Mode选项，可用于类或者方法上， 需要注意的是，这个注解的value是一个数组，可以把几种Mode集合在一起执行，还可以设置为Mode.All，即全部执行一遍。

2. @State
类注解，JMH测试类必须使用@State注解，State定义了一个类实例的生命周期，可以类比Spring Bean的Scope。由于JMH允许多线程同时执行测试，不同的选项含义如下：
> Scope.Thread：默认的State，每个测试线程分配一个实例；
> Scope.Benchmark：所有测试线程共享一个实例，用于测试有状态实例在多线程共享下的性能；
> Scope.Group：每个线程组共享一个实例；

3. @OutputTimeUnit
benchmark 结果所使用的时间单位，可用于类或者方法注解，使用java.util.concurrent.TimeUnit中的标准时间单位。

4. @Benchmark
方法注解，表示该方法是需要进行 benchmark 的对象。

5. @Setup
方法注解，会在执行 benchmark 之前被执行，正如其名，主要用于初始化。

6. @TearDown
方法注解，与@Setup 相对的，会在所有 benchmark 执行结束以后执行，主要用于资源的回收等。

7. @Param
成员注解，可以用来指定某项参数的多种情况。特别适合用来测试一个函数在不同的参数输入的情况下的性能。@Param注解接收一个String数组，在@setup方法执行前转化为为对应的数据类型。多个@Param注解的成员之间是乘积关系，譬如有两个用@Param注解的字段，第一个有5个值，第二个字段有2个值，那么每个测试方法会跑5*2=10次。

通过JMH来测试一下Java中几种常见的JSON解析库的性能

##### Gson
> 项目地址：https://github.com/google/gson

Gson是目前功能最全的Json解析神器，Gson当初是为因应Google公司内部需求而由Google自行研发而来，但自从在2008年五月公开发布第一版后已被许多公司或用户应用。Gson的应用主要为toJson与fromJson两个转换函数，无依赖，不需要例外额外的jar，能够直接跑在JDK上。在使用这种对象转换之前，需先创建好对象的类型以及其成员才能成功的将JSON字符串成功转换成相对应的对象。类里面只要有get和set方法，Gson完全可以实现复杂类型的json到bean或bean到json的转换，是JSON解析的神器。


##### FastJson
> 项目地址：https://github.com/alibaba/fastjson

Fastjson是一个Java语言编写的高性能的JSON处理器,由阿里巴巴公司开发。无依赖，不需要例外额外的jar，能够直接跑在JDK上。FastJson在复杂类型的Bean转换Json上会出现一些问题，可能会出现引用的类型，导致Json转换出错，需要制定引用。FastJson采用独创的算法，将parse的速度提升到极致，超过所有json库。

##### Jackson
> 项目地址：https://github.com/FasterXML/jackson

Jackson是当前用的比较广泛的，用来序列化和反序列化json的Java开源框架。Jackson社区相对比较活跃，更新速度也比较快， 从Github中的统计来看，Jackson是最流行的json解析器之一，Spring MVC的默认json解析器便是Jackson。

Jackson优点很多：

1. Jackson 所依赖的jar包较少，简单易用。
2. 与其他 Java 的 json 的框架 Gson 等相比，Jackson 解析大的 json 文件速度比较快。
3. Jackson 运行时占用内存比较低，性能比较好
4. Jackson 有灵活的 API，可以很容易进行扩展和定制。

目前最新版本是2.9.4，Jackson 的核心模块由三部分组成：

1. jackson-core 核心包，提供基于”流模式”解析的相关 API，它包括 JsonPaser 和 JsonGenerator。Jackson 内部实现正是通过高性能的流模式 API 的 JsonGenerator 和 JsonParser 来生成和解析 json。
2. jackson-annotations 注解包，提供标准注解功能；
3. jackson-databind 数据绑定包，提供基于”对象绑定” 解析的相关 API（ ObjectMapper ）和”树模型” 解析的相关 API（JsonNode）；基于”对象绑定” 解析的 API 和”树模型”解析的 API 依赖基于”流模式”解析的 API。

##### Json-lib
> http://json-lib.sourceforge.net/index.html

json-lib最开始的也是应用最广泛的json解析工具，json-lib 不好的地方确实是依赖于很多第三方包，对于复杂类型的转换，json-lib对于json转换成bean还有缺陷， 比如一个类里面会出现另一个类的list或者map集合，json-lib从json到bean的转换就会出现问题。json-lib在功能和性能上面都不能满足现在互联网化的需求。

![1563557790585](C:\Users\VULCAN\AppData\Roaming\Typora\typora-user-images\1563557790585.png)

从上面的测试结果可以看出，序列化次数比较小的时候，Gson性能最好，当不断增加的时候到了100000，Gson明细弱于Jackson和FastJson， 这时候FastJson性能是真的牛，另外还可以看到不管数量少还是多，Jackson一直表现优异。而那个Json-lib简直就是来搞笑的。

![1563557850653](C:\Users\VULCAN\AppData\Roaming\Typora\typora-user-images\1563557850653.png)

从上面的测试结果可以看出，反序列化的时候，Gson、Jackson和FastJson区别不大，性能都很优异，而那个Json-lib还是来继续搞笑的。



###  Java基础小测试

```
public class Horse {
    private Horse same;
    private String value;

    public Horse(String value) {
        this.value = value;
    }

    public Horse same(Horse horse){
        if (null != same){
            Horse same = horse;
            same.same = horse;
            same = horse.same;
        }
        return same.same;
    }

    public static void main(String[] args) {
        Horse horse = new Horse("horse");
        Horse cult = new Horse("cult");
        cult.same = cult;
        cult = cult.same(horse);
        System.out.println(horse.value);
        System.out.println(cult.value);
    }
}
```

父类、子类中的代码块执行顺序：父类静态代码块-子类静态代码块-主程序-父类非静态代码块-父类构造函数-子类非静态代码块-子类构造函数-一般方法（子类如果重写了父类的方法则执行子类方法体，否则执行父类方法体）。注意：父类和子类的静态代码块只执行一次，不论new多少个对象。

```
public class HelloA {
    public String name;
    public HelloA(){
        System.out.println("父类无参构造函数");
    }

    public HelloA(String name) {
        System.out.println("父类有参构造函数");
        this.name = name;
    }

    {
        System.out.println("父类非静态代码块");
    }
    static {
        System.out.println("父类静态代码块");
    }
    public void run(){
        System.out.println("父类方法体");
    }
}

public class HelloB extends HelloA {
    public HelloB() {
        System.out.println("子类无参构造函数");
    }

    public HelloB(String name) {
        super(name);
        System.out.println("子类有参构造函数");
    }

    {
        System.out.println("子类非静态代码块");
    }
    static {
        System.out.println("子类静态代码块");
    }
    
    public void run(){
        System.out.println("子类方法体");
    }

    public static void main(String[] args) {
        System.out.println("main start");
        new HelloB();
        new HelloB("张三").run();
        System.out.println("main end");
    }
}
```