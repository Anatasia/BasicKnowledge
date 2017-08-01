## 面向对象6大原则：

1）单一职责。一个类只负责一项职责，将一组相关性很高的函数、数据封装到一个类中。

2）开闭原则。对扩展开发，对修改封闭。

3）里氏替换原则。使用抽象和多态将设计中静态结构改为动态结构，维持设计的封闭性。任何基类可以出现的地方，子类一定可以出现。

4）依赖倒置原则。高层次的模块不应该依赖于低层次的模块，他们都应该依赖于抽象。抽象不应该依赖于具体的实现，具体实现应该依赖于抽象。

5）迪米特原则。又叫最少知识原则，一个对象应该对其他对象有尽可能少的了解。一个类应该对自己需要耦合或调用的类知道最少，不关心被耦合或调用类的内部实现，只负责调用提供的方法。

6）接口隔离原则。一个类对另一个类的依赖应该建立在最小的接口上。

# 1.单例模式

## 1）目的

SingleTon希望并限制该类的实例只能有一个。

## 2）实现方式

### a）Eager SingleTon

```
public class EagerSingleton {

    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {
    }

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```

这种称为“饿汉式”的实现方式可能是最简单也最常见的，顾名思义，该实例在类加载的时候就会自动创建不管之后是否被使用。所以如果该类实例化的开销比较大，这种方式或许就不太理想，不过它的优点也很明显，即无需担心多线程同步获取该实例时可能出现的并发问题。

### b）Lazy SingleTon

```
public class LazySingleton {

    private volatile static LazySingleton INSTANCE = null;

    private LazySingleton() {
    }

    public static synchronized LazySingleton getInstance() {
        if (INSTANCE == null)
            INSTANCE = new LazySingleton();
    return INSTANCE;
    }
}
```

这种方式也有个形象的名字“懒汉式”，既然觉得类加载时就完成实例化有点浪费，那不如将这一过程推迟到实际需要使用时，可是在此值得注意的是为了避免多线程并发场景下可能导致的莫名其妙多创建出一个实例的弊端，getInstance方法必须标记为synchronized方法或采用synchronized代码块来加锁实现。但是这种过度保护的代价是非常高昂的，其实只有当该实例未被创建时才有必要加锁控制并发，因此更多时候是没必要同步的，此类方式并不经济划算。

### c）Lazy Singleton with Double Check

```
public class LazySingletonWithDoubleCheck {

    private volatile static LazySingletonWithDoubleCheck INSTANCE = null;

    private LazySingletonWithDoubleCheck() {
    }

    public static LazySingletonWithDoubleCheck getInstance() {
        if (INSTANCE == null) {
            synchronized (LazySingletonWithDoubleCheck.class) {
                if (INSTANCE == null)
                    INSTANCE = new LazySingletonWithDoubleCheck();
            }
        }
        return INSTANCE;
    }
}
```

作为Lazy Singleton的改良版，这种采用了double-check的实现方式避免了对getInstance方法总是加锁。注意到尚未实例化时，存在两次检查的流程，第一次检查如果发现该实例已经存在就可以直接返回，否则则加类锁并进行第二次检查，原因在于可能出现多个线程同时通过了第一次检查，此时必须通过锁机制实现真正实例化时的排他性，保证只有一个线程成功抢占到锁并执行。此举即保证了线程安全，又将性能折损明显降低了，不失为比较理想的做法。

### d）Inner Class Singleton

```
public class InnerClassSingleton {

    private static class SingletonHolder {
        private static final InnerClassSingleton INSTANCE = new InnerClassSingleton();
    }

    private InnerClassSingleton() {
    }

    public static final InnerClassSingleton getInstance() {
        return SingletonHolder.INSTANCE;
}
}
```

另外一种可以有效解决线程并发问题并延迟实例化的做法就是如上代码所示的利用静态内部类来实现。单例类的实例被包裹在内部类中，因此该单例类加载时并不会完成实例化，直到有线程调用getInstance方法，内部类才会被加载并实例化单例类。这种做法应该说是比较令人满意的。

### 3）破坏单例

### a）Break Singleton with Clonable

```
public class ClonableSingleton implements Cloneable{

    private static final ClonableSingleton INSTANCE = new ClonableSingleton();

    private ClonableSingleton() {
    }

    public static ClonableSingleton getInstance() {
        return INSTANCE;
    }

    public Object clone() throws CloneNotSupportedException{
        return super.clone();
    }
}
```

Java中类通过实现Clonable接口并覆写clone方法就可以完成一个其对象的拷贝。而当Singleton类为Clonable时也自然无法避免可利用这种方式被重新创建一份实例。通过以下的测试代码即可检验通过clone我们可以有效破坏单例。

```
public static void checkClone() throws Exception {
    ClonableSingleton a = ClonableSingleton.getInstance();
    ClonableSingleton b = (ClonableSingleton) a.clone();

    assertEquals(a, b);
}
```

### b）Break Singleton with Serialization

```
public class SerializableSingleton implements Serializable{

    private static final long serialVersionUID = 6789088557981297876L;

    private static final SerializableSingleton INSTANCE = new SerializableSingleton();

    private SerializableSingleton() {
    }

    public static SerializableSingleton getInstance() {
        return INSTANCE;
    }
}
```

第二种破坏方式就是利用序列化与反序列化，当Singleton类实现了Serializable接口就代表它是可以被序列化的，该实例会被保存在文件中，需要时从该文件中读取并反序列化成 对象。可就是在反序列化这一过程中不知不觉留下了可趁之机，因为默认的反序列化过程是绕开构造函数直接使用字节生成一个新的对象。于是，Singleton在反序列化时被创造出第二个实例。通过如下代码可轻松实现这一行为，a与b最终并不相等。

```
public static void checkSerialization() throws Exception {
    File file = new File("serializableSingleton.out");
    ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(
    file));
    SerializableSingleton a = SerializableSingleton.getInstance();
    out.writeObject(a);
    out.close();

    ObjectInputStream in = new ObjectInputStream(new FileInputStream(file));
    SerializableSingleton b = (SerializableSingleton) in.readObject();
    in.close();

    assertEquals(a, b);
}
```

### c）Break Singleton with Reflection

```
public static void checkReflection() throws Exception {
    EagerSingleton a = EagerSingleton.getInstance();

    Constructor<EagerSingleton> cons = EagerSingleton.class
        .getDeclaredConstructor();
    cons.setAccessible(true);
    EagerSingleton b = (EagerSingleton) cons.newInstance();

    assertEquals(a, b);
}
```

前两种破坏方式说到底都是通过避免与私有构造函数正面冲突的方式另辟蹊径来实现的，而这种方式就显得艺高人胆大，既然你是私有的不允许外界直接调用，那么我就利用反射机制强行逼你就范：公开其访问权限。如此一来，原本看似安全的堡垒顷刻间沦为炮灰，Singleton再次沦陷。

### d）Break Singleton with Classloaders

```
public static void checkClassloader() throws Exception {
    String className = "fernando.lee.singleton.EagerSingleton";
    ClassLoader classLoader1 = new MyClassloader();
    Class<?> clazz1 = classLoader1.loadClass(className);

    ClassLoader classLoader2 = new MyClassloader();
    Class<?> clazz2 = classLoader2.loadClass(className);

    System.out.println("classLoader1 = " + clazz1.getClassLoader());
    System.out.println("classLoader2 = " + clazz2.getClassLoader());

    Method getInstance1 = clazz1.getDeclaredMethod("getInstance");
    Method getInstance2 = clazz2.getDeclaredMethod("getInstance");
    Object a = getInstance1.invoke(null);
    Object b = getInstance2.invoke(null);

    assertEquals(a, b);
}
```

Java中一个类并不是单纯依靠其全包类名来标识的，而是全包类名加上加载它的类加载器共同确定的。因此，只要是用不同的类加载器加载的Singleton类并不认为是相同的，因此单例会再次被破坏，通过自定义编写的MyClassLoader即可实现。

## 4）安全的单例模式

由此看来，Singleton唯有妥善关闭了如上所述的诸多后门才能称得上真正的单例。有两种方式可供参考，第一种方式通过完善现有实现让克隆、序列化、反射和类加载器无从下手，第二种方式则采取枚举类型间接实现单例。

### a）Safe Singleton

```
public class SafeSingleton implements Serializable, Cloneable {

    private static final long serialVersionUID = -4147288492005226212L;

    private static SafeSingleton INSTANCE = new SafeSingleton();

    private SafeSingleton() {
        if (INSTANCE != null) {
            throw new IllegalStateException("Singleton instance Already created.");
        }
    }

    public static SafeSingleton getInstance() {
        return INSTANCE;
    }

    private Object readResolve() throws ObjectStreamException {
        return INSTANCE;
    }

    public Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException("Singleton can't be cloned");
    }
}
```

在原有Singleton的基础上完善若干方法即可实现一个安全的更为纯正的Singleton。注意到当实例已经存在时试图通过调用私有构造函数会直接报错从而抵御了反射机制的入侵； 让调用clone方法直接报错避免了实例被克隆；覆写readReslove方法直接返回现有的实例本身可以防止反序列化过程中生成新的实例。

### b）Enum Singleton

```
public enum EnumSingleton{
    INSTANCE;

    private EnumSingleton(){
    }
}
```

采用枚举的方式实现Singleton非常简易，而且可直接通过EnumSingleton.INSTANCE获取该实例。Java中所有定义为enum的类内部都继承了Enum类，而Enum具备的特性包括类加载是静态的来保证线程安全，而且其中的clone方法是final的且直接抛出CloneNotSupportedException异常因而不允许拷贝，同时与生俱来的序列化机制也是直接由JVM掌控的并不会创建出新的实例，此外Enum不能被显式实例化反射破坏也不起作用。当然它也不是没有缺点，比如由于已经隐式继承Enum所以无法再继承其他类了（Java的单继承模式限制）。

# 2.观察者模式

观察者模式\(Observer Pattern\)：定义对象间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。观察者模式又叫做发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。

观察者模式需要的两个主体对象：被观察主体（Subject）、观察者（Observer），直观的说，观察者模式就是 被观察对象发生改变时能够通知观察者状态的变化 所以，观察者（Observer）依赖于被观察主体（Subject）的通知，在得到通知后进行后续的处理。

## 1\)UML类图

![](/assets/observer.png)

分析：

Subject：目标。他把所有对观察者对戏的引用保存在一个聚集里，每一个主题都可以有多个观察者。

Observer：观察者。为所有的具体观察者定义一个接口，在得到主题的通知时能够及时的更新自己。

ConcreteSubject：具体主题。将有关状态存入具体观察者对象。在具体主题发生改变时，给所有的观察者发出通知。

ConcreteObserver:具体观察者。实现抽象观察者角色所要求的更新接口，以便使本身的状态与主题状态相协调。

## 2）观察者模式实现

观察者模式又分为两种模式：push和pull。push是指suject在状态变化时将所有的状态信息都发给observer，pull则是suject通知observer更新时，observer获取自己感兴趣的状态。

两种模式在实现上的区别：push模式下，observer的update方法接收的是状态信息，而pull模式下，update方法接收的是suject对象，这种情况下，suject须提供状态信息的get方法，让observer可以获取自己感兴趣的信息。

## 3）观察者模式优缺点

### a）优点

1、当两个对象之间松耦合，他们依然可以交互，但是不太清楚彼此的细节。观察者模式提供了一种对象设计，让主题和观察者之间松耦合。主题所知道只是一个具体的观察者列表，每一个具体观察者都符合一个抽象观察者的接口。主题并不认识任何一个具体的观察者，它只知道他们都有一个共同的接口。

2、观察者模式支持“广播通信”。主题会向所有的观察者发出通知。

3、观察者模式符合“开闭原则”的要求。

### b）缺点

1、如果一个被观察者对象有很多的直接和间接的观察者的话，将所有的观察者都通知到会花费很多时间。

2、 如果在观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进  行循环调用，可能导致系统崩溃。

3、   观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化。

## 4）适用场景

1、一个抽象模型有两个方面，其中一个方面依赖于另一个方面。将这些方面封装在独立的对象中使它们可以各自独立地改变和复用。

2、一个对象的改变将导致其他一个或多个对象也发生改变，而不知道具体有多少对象将发生改变，可以降低对象之间的耦合度。

3、一个对象必须通知其他对象，而并不知道这些对象是谁。需要在系统中创建一个触发链，A对象的行为将影响B对象，B对象的行为将影响C对象……，可以使用观察者模式创建一种链式触发机制。

## 总结

1、观察者模式定义了对象之间的一对多关系。多个观察者监听同一个被观察者，当该被观察者的状态发生改变时，会通知所有的观察者。

2、观察者模式中包含四个角色。主题，它指被观察的对象。具体主题是主题子类，通常它包含有经常发生改变的数据，当它的状态发生改变时，向它的各个观察者发出通知；观察者，将对观察主题的改变做出反应；具体观察者中维护一个指向具体目标对象的引用，它存储具体观察者的有关状态，这些状态需要和具体目标的状态保持一致。

3、主题用一个共同的接口来更新观察者。

4、观察者与被观察者之间用松耦合方式结合。

5、有多个观察者时，不可以依赖特定的通知次序。

6、使用观察者模式，可以从被观察者处推或者拉数据。

## 4）观察者模式的应用

![](/assets/observer2.png)

### 主题接口

```
/**
 * 主题（发布者、被观察者）
  */
public interface Subject {

    /**
      * 注册观察者
      */
    void registerObserver(Observer observer);

    /**
     * 移除观察者
      */
    void removeObserver(Observer observer);

    /**
      * 通知观察者
      */
    void notifyObservers(); 
}

作者：张磊BARON
链接：http://www.jianshu.com/p/d55ee6e83d66
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

### **观察者接口**

```
/**
  * 观察者
  */
public interface Observer {
    void update();
}
```

### 公告牌用于显示的公共接口

```
public interface DisplayElement {
    void display();
}
```

```
public class WeatherData implements Subject {

    private List<Observer> observers;

    private float temperature;//温度
    private float humidity;//湿度
    private float pressure;//气压
    private List<Float> forecastTemperatures;//未来几天的温度

    public WeatherData() {
        this.observers = new ArrayList<Observer>();
    }

    @Override
    public void registerObserver(Observer observer) {
        this.observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        this.observers.remove(observer);
    }

    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update();
        }
    }

    public void measurementsChanged() {
        notifyObservers();
    }

    public void setMeasurements(float temperature, float humidity, 
    float pressure, List<Float> forecastTemperatures) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        this.forecastTemperatures = forecastTemperatures;
        measurementsChanged();
    }

    public float getTemperature() {
        return temperature;
    }

    public float getHumidity() {
        return humidity;
    }

    public float getPressure() {
        return pressure;
    }

    public List<Float> getForecastTemperatures() {
        return forecastTemperatures;
    }
}

作者：张磊BARON
链接：http://www.jianshu.com/p/d55ee6e83d66
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

```
public class CurrentConditionsDisplay implements Observer, DisplayElement {

    private WeatherData weatherData;

    private float temperature;//温度
    private float humidity;//湿度
    private float pressure;//气压

    public CurrentConditionsDisplay(WeatherData weatherData) {
        this.weatherData = weatherData;
        this.weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("当前温度为：" + this.temperature + "℃");
        System.out.println("当前湿度为：" + this.humidity);
        System.out.println("当前气压为：" + this.pressure);
    }

    @Override
    public void update() {
        this.temperature = this.weatherData.getTemperature();
        this.humidity = this.weatherData.getHumidity();
        this.pressure = this.weatherData.getPressure();
        display();
    }
}

作者：张磊BARON
链接：http://www.jianshu.com/p/d55ee6e83d66
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

```
public class ForecastDisplay implements Observer, DisplayElement {

    private WeatherData weatherData;

    private List<Float> forecastTemperatures;//未来几天的温度

    public ForecastDisplay(WeatherData weatherData) {
        this.weatherData = weatherData;
        this.weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("未来几天的气温");
        int count = forecastTemperatures.size();
        for (int i = 0; i < count; i++) {
            System.out.println("第" + i + "天:" + forecastTemperatures.get(i) + "℃");
        }
    }

    @Override
    public void update() {
        this.forecastTemperatures = this.weatherData.getForecastTemperatures();
        display();
    }
}

作者：张磊BARON
链接：http://www.jianshu.com/p/d55ee6e83d66
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

到这里，我们整个气象局的WeatherData应用就改造完成了。两个公告牌CurrentConditionsDisplay和ForecastDisplay实现了Observer和DisplayElement接口，在他们的构造方法中会调用WeatherData的registerObserver方法将自己注册成观察者，这样被观察者WeatherData就会持有观察者的应用，并将它们保存到一个集合中。当被观察者\`WeatherData状态发送变化时就会遍历这个集合，循环调用观察者公告牌更新数据的方法。后面如果我们需要增加或者删除公告牌就只需要新增或者删除实现了Observer和DisplayElement接口的公告牌就好了。

