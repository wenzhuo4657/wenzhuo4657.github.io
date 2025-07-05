+++
date = '2025-06-29T10:00:14+08:00'
draft = false
title = 'dubbo的spi'
layout = 'singlet'
+++





dubbo的spi相比于jdk的spi而言，提供了更为强大的功能，主要来说是帮助我们更好的面对多个服务互相依赖的场景，并且做了一定优化。（例如：按需加载）



## 按需加载



jdk的spi的配置文件

```
org.example.ToyotaCar
org.example.HondaCar
```

dubbo的spi的配置文件

```
toyota=org.example.ToyotaCar
honda=org.example.HondaCar
wrapper=org.example.aop.CarWrapper1
wrapper=org.example.aop.CarWrapper2
Race=org.example.ioc.RaceRes
red=org.example.ioc.PenRes

```

二者的区别在于dubbo中使用键值，可以实现按需加载，请注意，该加载并非指配置文件的加载，而加载配置文件之后的对于**服务对象的实例化**。





### 示例

#### jdk的spi

```
      ServiceLoader<Car> load = ServiceLoader.load(Car.class);

//        获取迭代器遍历
        Iterator<Car> iterator = load.iterator();
        while (iterator.hasNext()){
            Car registry = iterator.next();
            registry.run();
        }
        
```

追溯源码可以看到迭代器内部。

![image-20250122174648892](https://blog.wenzhuo4657.org/img/image-20250122174648892.png)

关键成员变量

Iterator<String> pending ;//配置文件读取的数据  **该变量也是迭代器**
String nextName ;//下一个将要读取的配置



存在关键方法

hasNextService：用于判断下一个服务名称，即配置文件当中的全限定类名。

```
       private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();//获取下一个元素，
            return true;
        }

```



nextService：

```
    private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);//获取类加载器
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());//实例化对象
                providers.put(cn, p);//延迟加载，将其收入到一个map集合当中
                return p;//返回加载元素
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
```





#### dubbo的spi

dubbo的spi则使用键值加载。

```
    ExtensionLoader<Car> extensionLoader;
    @Before
    public void  fi(){
        extensionLoader = ExtensionLoader.getExtensionLoader(Car.class);
    }


    /**
     *  @author:wenzhuo4657
        des:
    dubbo基本服务发现，和jdk的主要区别在于可以用键去获取指定的扩展实现。
    */
        @Test
        public void test(){
            Car car = extensionLoader.getExtension("ali",false);//false表示不进行自动装配等其他配置，默认为true,
            car.run();
            if (car instanceof HondaCar){
                System.out.println("true");
            }
        }
       
```



追溯getExtension方法，可以看到关键实例化对象的createExtension方法

```
 private T createExtension(String name, boolean wrap) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);


            if (wrap) {

                List<Class<?>> wrapperClassesList = new ArrayList<>();
                if (cachedWrapperClasses != null) {
                    wrapperClassesList.addAll(cachedWrapperClasses);
                    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                    Collections.reverse(wrapperClassesList);
                }

                if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                    for (Class<?> wrapperClass : wrapperClassesList) {
                        Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                        if (wrapper == null
                                || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                        }
                    }
                }
            }

            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```



1，提前获取类加载器，

Class<?> clazz = getExtensionClasses().get(name);方法内部可以看到有一个成员变量`private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();`





# DUBBO的扩展点加载

[扩展点加载 | Apache Dubbo](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/spi/):该文档旨在对官网翻译，助于理解



dubbo扩展点加载与单纯的spi不同，会使用Wrapper包装类对服务方法进行扩展，实际上将，这有点像aop。

包装之后，获取的对象将不是所需要的服务对象，而是包装类对象。

```
public class XxxProtocolWrapper implements Protocol {
    Protocol impl;
 
    public XxxProtocolWrapper(Protocol protocol) { impl = protocol; }
 
    // 接口方法做一个操作后，再调用extension的方法
    public void refer() {
        //... 一些操作
        impl.refer();
        // ... 一些操作
    }
 
    // ...
}

```



可以在加载是通过参数指定是否开启包装类代理。

```
Car car = extensionLoader.getExtension("ali",false);//false表示不进行自动装配等其他配置，默认为true,
```



## 自动装配

自动装配对应spring DI自动注入，判断方式为方法名称。

形如setWheelMaker 方法的 WheelMaker 也是扩展点则会注入 WheelMaker 的实现。



````
public class RaceCarMaker implements CarMaker {
    WheelMaker wheelMaker;
 
    public void setWheelMaker(WheelMaker wheelMaker) {
        this.wheelMaker = wheelMaker;
    }
 
    public Car makeCar() {
        // ...
        Wheel wheel = wheelMaker.makeWheel();
        // ...
        return new RaceCar(wheel, ...);
    }
}
````

但是这种注入无法指定依赖服务的实现，一般为默认实现或者优先级判断使用哪个实现，但无论是哪种都是全局唯一的默认，无法实现特例。





## 自适应装配



该装配利用了包装类和方法参数，实现在扩展时加载服务对象并调用方法。

（**该方式和自动装配并不冲突，只是并不使用自动装配的服务，并且由于包装类方法执行的缘故，甚至不用判断是否使用默认服务。**）

```
public interface CarMaker {
	 @Adaptive({"CarMaker"})
    Car makeCar(URL url);
}
 
public interface WheelMaker {
	 @Adaptive({"WheelMaker"})
    Wheel makeWheel(URL url);
}
```



```
public class RaceCarMaker implements CarMaker {
    WheelMaker wheelMaker;
 
    public void setWheelMaker(WheelMaker wheelMaker) {
        this.wheelMaker = wheelMaker;
    }
 
    public Car makeCar(URL url) {
        // ...
        Wheel wheel = wheelMaker.makeWheel(url);
        // ...
        return new RaceCar(wheel, ...);
    }
}
```

```
        Pen pen = extensionLoader.getAdaptiveExtension();
        URL url = new URL("dubbo", "127.0.0.1", 20880).
                addParameter("WheelMaker", "wheel").
                addParameter("CarMaker","car");

        // 调用方法
        pen.run(url);
```

使用@Adaptive注解定义包装类实例化服务的key,在url参数中指定，最终实现特定服务调用特定服务。

深入源码中即可发现这一实现的依赖于包装类。

```
public java.lang.String queryCountry(org.apache.dubbo.common.URL arg0) {         if (arg0 == null) throw new IllegalArgumentException("url == null");         org.apache.dubbo.common.URL url = arg0;         String extName = url.getParameter("person.service", "china");//获取指定扩展服务         if (extName == null)             throw new IllegalStateException("Failed to get extension (org.dubbo.spi.example.PersonService) name from url (" + url.toString() + ") use keys([person.service])");         org.dubbo.spi.example.PersonService extension = (org.dubbo.spi.example.PersonService) ExtensionLoader.getExtensionLoader(org.dubbo.spi.example.PersonService.class).getExtension(extName); //扩展服务加载      
return extension.queryCountry(arg0);//扩展服务执行    
}
```

它会在扩展执行时通过url参数中的指定值，加载扩展服务执行。 







# 补充要点



## jdk的加载默认使用无参构造器



这一点可以在ServiceLoader.LazyIterator#nextService()中看到

`S p = service.cast(c.newInstance());`



而Dubbo可以通过url携带参数实例化。


```

```
