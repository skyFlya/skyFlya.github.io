---
title: Android AOP - JDK动态代理
tags:
  - AOP
categories: Android
updated: 1646918843000
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLrVJK.md.png'
date: 2022-03-10 21:25:00
---


JDK的动态代理底层是通过Java反射机制实现的，并且需要目标对象继承自一个接口才能生成它的代理类
<!--more-->

####  Java动态代理

>  JDK 运行期间，动态创建 class 字节码并加载到 JVM

1. JDK的动态代理需要实现一个处理方法调用的Handler，用于实现代理方法的内部逻辑，实现InvocationHandler接口
```java
public class JdkProxyHandler implements InvocationHandler {
    private Object target;
    public JdkProxyHandler(Object target){
        this.target = target;
    }
    // proxy：生成的代理对象, obj：目标方法, args：目标方法参数
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("处理前");
        Object result = method.invoke(target,args);
        System.out.println("处理后");
        return result;
    }
}
```
2. 代理
```java
//2-1
public interface ISender {
    public boolean send();
}
//2-2
public class SmsSender implements ISender {
    public boolean send() {
        System.out.println("sending msg");
        return true;
    }
}
//2-3
@Test
public void testJdkProxy(){
    ISender sender = (ISender) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(),
            new Class[]{ISender.class},
            new JdkProxyHandler(new SmsSender()));
    boolean result = sender.send();
    System.out.println("代理对象：" + sender.getClass().getName());
    System.out.println("输出结果：" + result);
}
```


通 过 JDK 的 java.lang.reflect.Proxy 类实现动态代理 ， 会使用其静态方法newProxyInstance()，依据目标对象、业务接口及调用处理器三者，自动生成一个动态代理对象。

`public static newProxyInstance ( ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)`
- loader：目标类的类加载器，通过目标对象的反射可获取
- interfaces：目标类实现的接口数组，通过目标对象的反射可获取
- handler：调用处理器。

##### 原理

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) throws IllegalArgumentException {
    //1.检查
    Objects.requireNonNull(h);
    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    //查找或生成指定的代理类
    Class<?> cl = getProxyClass0(loader, intfs);
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
        //通过反射创建代理对象
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

### 局限性

只能代理接口，并且只能修改接口声明的方法

### Refer

[Java动态代理四种实现方式详解](https://www.jb51.net/article/209607.htm)
[Java 动态代理 - 王陸](https://www.cnblogs.com/wkfvawl/p/15030814.html)