面向对象6大原则：

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
