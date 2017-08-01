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

a）Safe Singleton

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



