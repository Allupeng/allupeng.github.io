---

layout: post

titile: "反射"

data: 2021-05-19 20:32:15 +0800

categories: 计算机

tags: Java

---

## 反射的概念

​	首先，**反射**对立的就是正射。我们常用的new就是产生一个实例，也就是正射。

​	假设我们最近有个需要需要使用HashMap，那么我们通过以下来实例一个HashMap。

```java
  Map<Object,Object> map = new HashMap<>();
```

​	那么如果要求突然变化，需要我们使用一个Hashtable怎么办呢？我们需要去修改代码来满足目的。

```java
 Map<Object,Object> map = new Hashtable<>();
```

​	如果要求又变更为HashMap呢？那么我们又得将代码改回原来的样子。

```java
        Map<Object,Object> map = new HashMap<>();
      //  Map<Object,Object> map = new Hashtable<>();
```

​	大家应该发现了，这样非常麻烦。得**修改代码、重新打包、编译**。那么我们如何解决这个呢？首先，我们可以想到用一个函数根据**传入的参数来自动创建实例**。

```java
    public <T> Map<T,T> productList(String clazz){
        Map<T,T> map = null;
        if (clazz.equals("HashMap"))
            map = new HashMap<>();
        else if (clazz.equals("Hashtable"))
            map = new Hashtable<>();
        
        return map;
    }
```

​	我们通过传入参数**clazz**来构建自己想要的List，那么如果我们之后还想要其他构建其他的呢？那么我们不免又要修改代码。那么怎么办呢？

​	我们可以通过**反射**来进行类的创建。在代码运行前，我们**不确定**将来会用什么数据结构和类、而**反射**可以在**程序运行过程**中动态的**获取类信息**和**调用类方法**。通过反射构造函数如下所示：

```java
    public <T> Map<T,T> productList(String clazz) throws Exception{
        Map<T,T> map = null;
        Class cl = Class.forName(clazz);
        Constructor con = cl.getConstructor();
        map = (Map<T, T>) con.newInstance();
        return map;
    }
```

​	无论什么类，只要**实现了Map接口**，我们可以通过**类名的全路径**来创建一个实例。即使我们想创建TreeMap或者ConcurrentHashMap，我们也不需要修改源代码。这样是不是方便多了？