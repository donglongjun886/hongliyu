---
title: JDK动态代理实现原理
date: 2018-03-06 17:23:28
tags: [代理模式,Java,Spring]
categories: 设计模式
---
# 前言
Spring AOP使用JDK动态代理或者CGLIB来为目标对象创建代理。在DefaultAopProxyFactory类中是这样创建AOP代理的：
```java
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
# 静态代理
```java
public class StaticProxy {
    public interface Calculate{
        public void doSomething();
    }
    public static class CalculateImpl implements Calculate{
        public void doSomething() {
            System.out.println("do Something");
        }    
    }
    public static class Proxy implements Calculate{ 
        private Calculate target;
        public Proxy(Calculate target) {
            this.target=target;
        }
        public void doSomething() {
           System.out.println("before");
           target.doSomething();
           System.out.println("after");
        }
    }
    public static void main(String[] args) {
        Calculate calculator=new Proxy(new CalculateImpl());
        calculator.doSomething();
    }
}
```
静态代理的缺点就是要事先生成代理类，实现代理接口中的每一个方法，在某些情况下工作量巨大。
# 动态代理
```java
public class MyInvocationHandler implements InvocationHandler{ 
    private Object target;
    public MyInvocationHandler(Object target) {
        super();
        this.target=target;
    }
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before");
        Object result=method.invoke(target, args);
        System.out.println("after");
        return result;
    }
    public Object getProxy() {
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), target.getClass().getInterfaces(), this);
    }
}
```
## InvocationHandler接口
InvocationHandler接口很简单，只有一个invoke方法。
```java
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
```
每一个代理实例的调用处理器（invocation handler）都应该实现InvocationHandler接口，每个代理实例都有关联的调用处理器。当代理实例上的方法被调用时，方法调用会被编码并分派给对应调用处理器的invoke方法。
先写个测试样例如下：
```java
public class ProxyTest {
    public  interface UserService{
        public void doSomething();
    }
    public class UserServiceImpl implements UserService{
        public void doSomething() {
            System.out.println("userServiceImpl do something");
        }  
    }
    @Test
    public void test() {
        UserService userService=new UserServiceImpl();
        MyInvocationHandler invocationHandler=new MyInvocationHandler(userService);
        UserService proxy=(UserService)invocationHandler.getProxy();
        proxy.doSomething();
    }
}
```
## 代理的实例化
创建实例需要三个参数：定义代理类的calssLoad、代理类实现的接口列表和分派方法调用的调用处理器。
```java
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) {
        Objects.requireNonNull(h);

        final Class<?> caller = System.getSecurityManager() == null
                                    ? null
                                    : Reflection.getCallerClass();

        /*
         * Look up or generate the designated proxy class and its constructor.
         */
        Constructor<?> cons = getProxyConstructor(caller, loader, interfaces);

        return newProxyInstance(caller, cons, h);
    }
 ```
 newProxyInstance有四个步骤：
 * 校验调用处理器h是否为空
 * 反射获得调用类caller
 * 获得构造器cons
 * 调用newProxyInstance方法

```java
    private static Constructor<?> getProxyConstructor(Class<?> caller,
                                                      ClassLoader loader,
                                                      Class<?>... interfaces)
    {
        // optimization for single interface
        if (interfaces.length == 1) {
            Class<?> intf = interfaces[0];
            if (caller != null) {
                checkProxyAccess(caller, loader, intf);
            }
            return proxyCache.sub(intf).computeIfAbsent(
                loader,
                (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()
            );
        } else {
            // interfaces cloned
            final Class<?>[] intfsArray = interfaces.clone();
            if (caller != null) {
                checkProxyAccess(caller, loader, intfsArray);
            }
            final List<Class<?>> intfs = Arrays.asList(intfsArray);
            return proxyCache.sub(intfs).computeIfAbsent(
                loader,
                (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()
            );
        }
    }
```
checkProxyAccess检查创建代理类的权限，关于这部分权限源码有这样一段话：
>  If an interface is non-public, the proxy class must be defined by the defining loader of the interface.  If the caller's class loader is not the same as the defining loader of the interface, the VM will throw IllegalAccessError when the generated proxy class is being defined.

computeIfAbsent方法从本地缓存map查询，若无结果则新建Memoizer并用Memoizer的get方法返回值更新map。在get方法中有这样一句话：
```java
this.v = v = Objects.requireNonNull(
              mappingFunction.apply(cl, clv));
 ```

mappingFunction即之前computeIfAbsent的第二个参数。最终从build方法取得了代理类的构造器。
```java
        Constructor<?> build() {
            Class<?> proxyClass = defineProxyClass(module, interfaces);
            final Constructor<?> cons;
            try {
                cons = proxyClass.getConstructor(constructorParams);
            } catch (NoSuchMethodException e) {
                throw new InternalError(e.toString(), e);
            }
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
            return cons;
        }
```
上面constructorParams的值就是InvocationHandler。
```java
    /** parameter types of a proxy class constructor */
    private static final Class<?>[] constructorParams =
        { InvocationHandler.class };
 ```
在defineProxyClass方法中ProxyGenerator.generateProxyClass(..)生成了字节流，在saveGeneratedFiles为true的情况下保存为class文件。
```java
    static byte[] generateProxyClass(final String name,
                                     Class<?>[] interfaces,
                                     int accessFlags)
    {
        ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
        final byte[] classFile = gen.generateClassFile();
        if (saveGeneratedFiles) {
            java.security.AccessController.doPrivileged(
            new java.security.PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        int i = name.lastIndexOf('.');
                        Path path;
                        if (i > 0) {
                            Path dir = Paths.get(name.substring(0, i).replace('.', File.separatorChar));
                            Files.createDirectories(dir);
                            path = dir.resolve(name.substring(i+1, name.length()) + ".class");
                        } else {
                            path = Paths.get(name + ".class");
                        }
                        Files.write(path, classFile);
                        return null;
                    } catch (IOException e) {
                        throw new InternalError(
                            "I/O exception saving generated file: " + e);
                    }
                }
            });
        }
        return classFile;
    }
```
generateClassFile具体有三个步骤：
* Assemble ProxyMethod objects for all methods to generate proxy dispatching code for.
* Assemble FieldInfo and MethodInfo structs for all of fields and methods in the class we are generating.
* Write the final class file.

在第一步中会添加代理接口的所有方法
```java
        for (Class<?> intf : interfaces) {
            for (Method m : intf.getMethods()) {
                addProxyMethod(m, intf);
            }
        }
```
第二步会将所有方法信息添加到一个list中去,包括构造函数、代理方法和静态初始化方法
```java
            methods.add(generateConstructor());

            for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
                for (ProxyMethod pm : sigmethods) {

                    // add static field for method's Method object
                    fields.add(new FieldInfo(pm.methodFieldName,
                        "Ljava/lang/reflect/Method;",
                         ACC_PRIVATE | ACC_STATIC));

                    // generate code for proxy method and add it
                    methods.add(pm.generateMethod());
                }
            }

            methods.add(generateStaticInitializer());
```
第三步依次将信息写入DataOutputStream流中，包括魔数、版本号、接口、值域、方法等
在运行时添加参数-Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true就可得到generateProxyClass生成的class文件。反编译class文件可得到：
```java
package com.sun.proxy;

import java.lang.reflect.UndeclaredThrowableException;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import saber.designPatterns.Proxy.ProxyTest$UserService;
import java.lang.reflect.Proxy;

public final class $Proxy7 extends Proxy implements ProxyTest$UserService
{
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;
    
    public $Proxy7(final InvocationHandler invocationHandler) {
        super(invocationHandler);
    }
    
    public final boolean equals(final Object o) {
        try {
            return (boolean)super.h.invoke(this, $Proxy7.m1, new Object[] { o });
        }
        catch (Error | RuntimeException error) {
            throw;
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
    
    public final void doSomething() {
        try {
            super.h.invoke(this, $Proxy7.m3, null);
        }
        catch (Error | RuntimeException error) {
            throw;
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
    
    public final String toString() {
        try {
            return (String)super.h.invoke(this, $Proxy7.m2, null);
        }
        catch (Error | RuntimeException error) {
            throw;
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
    
    public final int hashCode() {
        try {
            return (int)super.h.invoke(this, $Proxy7.m0, null);
        }
        catch (Error | RuntimeException error) {
            throw;
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
    
    static {
        try {
            $Proxy7.m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            $Proxy7.m3 = Class.forName("saber.designPatterns.Proxy.ProxyTest$UserService").getMethod("doSomething", (Class<?>[])new Class[0]);
            $Proxy7.m2 = Class.forName("java.lang.Object").getMethod("toString", (Class<?>[])new Class[0]);
            $Proxy7.m0 = Class.forName("java.lang.Object").getMethod("hashCode", (Class<?>[])new Class[0]);
        }
        catch (NoSuchMethodException ex) {
            throw new NoSuchMethodError(ex.getMessage());
        }
        catch (ClassNotFoundException ex2) {
            throw new NoClassDefFoundError(ex2.getMessage());
        }
    }
}
```
在newProxyInstance中cons调用了newInstance方法实例化了代理对象
```java
    private static Object newProxyInstance(Class<?> caller, // null if no SecurityManager
                                           Constructor<?> cons,
                                           InvocationHandler h) {
        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (caller != null) {
                checkNewProxyPermission(caller, cons.getDeclaringClass());
            }

            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException | InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        }
    }
```








