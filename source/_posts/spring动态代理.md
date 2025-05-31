---
title: Spring动态代理全解析
date: 2023-02-24 11:12:11
categories: 
	- Spring
	- Java
tags: 
    - aop
    - llm
    - spring
toc: true
description: "Spring动态代理全解析：生成时机、源码脉络与代理类形态"
---

## Spring动态代理全解析：生成时机、源码脉络与代理类形态

Spring框架的强大功能很大程度上依赖于其AOP（面向切面编程）机制，而动态代理则是AOP的核心实现方式。理解Spring何时以及如何为Bean生成代理，对于深入掌握Spring至关重要。

### 一、代理类生成的时机：Bean生命周期中的关键节点

Spring Bean的生命周期是一个复杂但有序的过程。代理类的生成通常发生在Bean**初始化阶段之后**的一个特定步骤。

核心的角色是 `BeanPostProcessor` 接口。这个接口允许我们在Bean的初始化前后插入自定义的逻辑。Spring AOP正是通过实现这个接口（具体来说是其子类，如 `AbstractAutoProxyCreator`）来在合适的时机创建代理对象的。

**关键时机点：`BeanPostProcessor`的`postProcessAfterInitialization`方法**

1. **实例化 (Instantiation)**：Spring根据Bean定义创建Bean的原始实例。
2. **属性填充 (Populate properties)**：Spring为Bean实例注入依赖的属性。
3. **初始化 (Initialization)**：
   - 执行各种Aware接口的回调（如 `BeanNameAware`, `BeanFactoryAware`）。
   - 执行 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法。
   - 如果Bean实现了 `InitializingBean` 接口，执行其 `afterPropertiesSet()` 方法。
   - 执行Bean定义中指定的自定义 `init-method`。
   - **执行 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法**：
     - **这是AOP代理创建的主要发生地**。
     - Spring容器中注册的 `AbstractAutoProxyCreator`（例如处理 `@AspectJ` 注解的 `AnnotationAwareAspectJAutoProxyCreator`）会在这里检查当前的Bean是否需要被代理。
     - 如果判断需要代理（比如该Bean的方法匹配了某个切点，或者Bean上有 `@Transactional` 等需要代理的注解），`AbstractAutoProxyCreator` 就会为原始Bean实例创建一个代理对象，并**返回这个代理对象**。
     - 此时，Spring容器后续管理和注入的就不再是原始的Bean实例，而是这个代理实例了。

所以，简单来说，当一个普通的Bean经历完标准的实例化、属性填充和初始化方法（如`afterPropertiesSet`或自定义`init-method`）之后，Spring AOP的“代理检察官”（`BeanPostProcessor`）会介入，看看是否需要给这个刚“出炉”的Bean套上一层代理“外套”。如果需要，就生成并返回代理；如果不需要，就返回原始Bean。

### 二、Bean加载与代理生成源码脉络（简化版）

理解源码需要一定的耐心，我们尝试梳理一个简化的调用链，让你了解大致流程：

1. **获取Bean的入口：`getBean()`**
   - 当你调用 `ApplicationContext.getBean("someBean")` 时，最终会调用到 `AbstractBeanFactory` 的 `getBean()` 方法。
2. **核心处理：`doGetBean()`**
   - `AbstractBeanFactory.doGetBean()` 是获取Bean的核心逻辑。它会处理单例、原型等不同作用域的Bean。
   - 对于单例Bean，它会先尝试从缓存（`singletonObjects`）中获取。如果获取不到，则进入创建流程。
3. **创建Bean实例：`createBean()`**
   - 如果Bean需要被创建，会调用 `AbstractAutowireCapableBeanFactory.createBean()`。
4. **实际创建Bean：`doCreateBean()`**
   - `AbstractAutowireCapableBeanFactory.doCreateBean()` 负责Bean的实例化（`createBeanInstance`）和属性填充（`populateBean`）。
5. **初始化Bean：`initializeBean()`**
   - 在Bean实例化和属性填充完毕后，会调用 `AbstractAutowireCapableBeanFactory.initializeBean()`。这个方法是初始化Bean的关键，也是代理可能产生的地方。
   - `initializeBean()` 方法内部会依次调用：
     - `applyBeanPostProcessorsBeforeInitialization()`：执行所有 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法。
     - `invokeInitMethods()`：执行 `InitializingBean.afterPropertiesSet()` 和自定义的 `init-method`。
     - **`applyBeanPostProcessorsAfterInitialization()`**：执行所有 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法。**这是我们的主角登场的地方！**
6. **AOP代理的创建：`AbstractAutoProxyCreator.postProcessAfterInitialization()`**
   - 在 `applyBeanPostProcessorsAfterInitialization()` 中，Spring会遍历所有注册的 `BeanPostProcessor`。
   - 当轮到 `AbstractAutoProxyCreator`（或其子类，如 `AnnotationAwareAspectJAutoProxyCreator`）时，它的 `postProcessAfterInitialization(Object bean, String beanName)` 方法会被调用。
   - 这个方法的核心逻辑是：
     - 判断当前 `bean` 是否已经被代理过，或者是否是基础设施类（不需要代理）。
     - 调用 `wrapIfNecessary(bean, beanName, cacheKey)` 方法。
7. **`wrapIfNecessary()`：决定是否及如何代理**
   - `AbstractAutoProxyCreator.wrapIfNecessary()` 方法是实际决定是否创建代理以及如何创建代理的地方。
   - 它会收集所有适用于当前Bean的**通知器 (Advisors)**。Advisors封装了切面中的通知（Advice）和切点（Pointcut）。
   - 如果找到了适用于该Bean的Advisors（意味着这个Bean需要被AOP增强），它就会调用 `createProxy()` 方法。
8. **`createProxy()`：使用ProxyFactory创建代理**
   - `AbstractAutoProxyCreator.createProxy()` 方法会：
     - 创建一个 `ProxyFactory` 实例。
     - 将目标Bean（`targetSource`）、需要应用的Advisors、目标Bean实现的接口等信息设置到 `ProxyFactory` 中。
     - `ProxyFactory` 会根据配置（目标类是否实现接口、`proxyTargetClass`属性等）决定使用JDK动态代理还是CGLIB代理。
     - 调用 `proxyFactory.getProxy(getProxyClassLoader())` 来实际生成并返回代理对象。
9. **返回代理对象**
   - `AbstractAutoProxyCreator.postProcessAfterInitialization()` 方法最终会返回 `createProxy()` 生成的代理对象（如果需要代理的话），或者原始的 `bean` 对象（如果不需要代理）。
   - 这个返回的对象（可能是原始对象，也可能是代理对象）会被放入Spring容器的单例缓存中，后续所有对该Bean的请求都会得到这个对象。

**简化流程总结：**

`getBean()` -> `doGetBean()` -> (缓存未命中) -> `createBean()` -> `doCreateBean()` (实例化、属性填充) -> `initializeBean()` -> `applyBeanPostProcessorsAfterInitialization()` -> `AbstractAutoProxyCreator.postProcessAfterInitialization()` -> `wrapIfNecessary()` -> (如果需要代理) -> `createProxy()` (使用`ProxyFactory`) -> 返回代理对象。

### 三、生成的代理类是什么样子？

Spring生成的代理类在运行时动态创建，我们无法直接看到它们的Java源码文件，但可以通过一些特征和工具（如Debug模式下的变量视图，或使用Java反编译工具分析内存中的类）来理解它们的结构。

#### 1. JDK动态代理生成的代理类

- **条件**：当目标Bean**实现了一个或多个接口**，并且Spring没有被强制要求使用CGLIB时（即`proxyTargetClass`属性为`false`，这是默认情况）。
- **特征**：
  - **命名规则**：通常类名类似于 `com.sun.proxy.$ProxyX`，其中 `X` 是一个数字，如 `$Proxy0`, `$Proxy12`。
  - **实现接口**：这个代理类会实现目标Bean所实现的所有接口，以及一些Spring内部的标记接口（如 `org.springframework.aop.SpringProxy`, `org.springframework.aop.framework.Advised`）。
  - **继承关系**：它继承自 `java.lang.reflect.Proxy` 类。
  - **内部结构**：
    - 它不包含目标Bean的原始业务逻辑代码。
    - 它内部持有一个 `java.lang.reflect.InvocationHandler` 接口的实例。在Spring中，这个`InvocationHandler`的具体实现通常是 `org.springframework.aop.framework.JdkDynamicAopProxy`。
    - 当代理对象的任何一个接口方法被调用时，这个调用会被转发到 `JdkDynamicAopProxy` 的 `invoke()` 方法。
  - **`JdkDynamicAopProxy.invoke()` 的工作**：
    - 获取与当前方法匹配的所有AOP通知（Advisors/Interceptors）。
    - 构建一个调用链（Interceptor chain）。
    - 依次执行调用链中的通知（例如，`@Before`通知 -> 目标方法 -> `@AfterReturning`通知）。
    - 最终，如果需要，会通过反射调用原始目标Bean的对应方法。

**举例：**

假设我们有：

```
// 接口
public interface UserService {
    void addUser(String username);
    String getUser(String username);
}

// 实现类
public class UserServiceImpl implements UserService {
    public void addUser(String username) {
        System.out.println("UserServiceImpl: Adding user " + username);
    }
    public String getUser(String username) {
        System.out.println("UserServiceImpl: Getting user " + username);
        return "User: " + username;
    }
}

// 切面 (简单日志)
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.UserService.addUser(..))")
    public void logBeforeAddUser(JoinPoint joinPoint) {
        System.out.println("LoggingAspect: Before adding user - " + joinPoint.getArgs()[0]);
    }
}
```

如果Spring为 `UserServiceImpl` 生成JDK动态代理，这个代理类（假设叫 `$Proxy12`）在概念上会像这样：

```
// 伪代码，实际是运行时生成的字节码
public final class $Proxy12 extends java.lang.reflect.Proxy implements UserService, SpringProxy, Advised {
    private static Method m3; // addUser(String)
    private static Method m4; // getUser(String)
    // ... 其他接口的方法引用

    // 构造函数，传入 InvocationHandler
    public $Proxy12(InvocationHandler h) {
        super(h);
    }

    // 实现UserService接口的方法
    public final void addUser(String var1) {
        try {
            // this.h 就是 JdkDynamicAopProxy 实例
            // 调用会被转发到 JdkDynamicAopProxy.invoke(this, m3, new Object[]{var1})
            this.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error e) {
            throw e;
        } catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }

    public final String getUser(String var1) {
        try {
            return (String) this.h.invoke(this, m4, new Object[]{var1});
        } catch (RuntimeException | Error e) {
            throw e;
        } catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
    // ... 实现 SpringProxy, Advised 等接口的方法
}
```

当你调用 `$Proxy12.addUser("test")` 时，实际上是调用了 `JdkDynamicAopProxy` 实例的 `invoke` 方法。这个 `invoke` 方法会先执行 `LoggingAspect` 中的 `logBeforeAddUser` 通知，然后再调用原始 `UserServiceImpl` 实例的 `addUser` 方法。

#### 2. CGLIB代理生成的代理类

- **条件**：

  - 当目标Bean**没有实现任何接口**时。
  - 或者，当Spring被明确配置为对类进行代理时（即`proxyTargetClass`属性设置为`true`）。

- **特征**：

  - **命名规则**：类名通常是目标类的名称加上 

    EnhancerBySpringCGLIB

     或类似的后缀，并附带一个哈希码，例如 `com.example.UserServiceImpl`

    EnhancerBySpringCGLIB

    `a1b2c3d4`。

  - **继承关系**：这个代理类会**继承**目标Bean的类。因此，目标类不能是 `final` 的，需要被代理的方法也不能是 `final` 或 `private` 的。

  - **实现接口**：它也会实现一些Spring内部的标记接口，如 `SpringProxy`, `Advised`。

  - **内部结构**：

    - 它会重写目标类中所有公共的、受保护的非 `final` 方法。
    - 在重写的方法内部，它不会直接包含原始业务逻辑，而是会委托给CGLIB的回调机制。
    - Spring会设置一个或多个CGLIB的 `MethodInterceptor`（通常是 `org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor`）作为回调。

  - **`DynamicAdvisedInterceptor.intercept()` 的工作**：

    - 与JDK代理中的`invoke`方法类似，当代理对象的被重写方法被调用时，会触发 `DynamicAdvisedInterceptor` 的 `intercept()` 方法。
    - 这个方法负责获取匹配的通知、构建调用链、执行通知，并最终调用原始目标Bean的对应方法（通过 `MethodProxy.invokeSuper()` 调用父类，即原始类的方法）。

**举例：**

如果 `UserServiceImpl` 没有实现 `UserService` 接口（或者 `proxyTargetClass=true`），Spring会为其生成CGLIB代理。这个代理类（假设叫 `UserServiceImpl`

EnhancerBySpringCGLIB

`a1b2c3d4`）在概念上会像这样：

```
// 伪代码，实际是运行时生成的字节码
public class UserServiceImpl$$EnhancerBySpringCGLIB$$a1b2c3d4 extends UserServiceImpl implements SpringProxy, Advised {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static Callback[] CGLIB$STATIC_CALLBACKS; // 其中一个就是 DynamicAdvisedInterceptor
    // ... 其他CGLIB内部字段和方法

    // 重写的 addUser 方法
    public final void addUser(String var1) {
        // CGLIB$CALLBACK_0 就是我们关心的 DynamicAdvisedInterceptor
        MethodInterceptor interceptor = CGLIB$CALLBACK_0;
        if (interceptor == null) {
            // ... CGLIB内部初始化逻辑 ...
            super.addUser(var1); // 如果没有拦截器，直接调用父类（原始类）方法
            return;
        }
        // 调用拦截器的 intercept 方法
        // MethodProxy 用于高效调用父类（原始类）的同名方法
        interceptor.intercept(this, /* addUser方法引用 */, new Object[]{var1}, /* MethodProxy */);
    }

    // 重写的 getUser 方法
    public final String getUser(String var1) {
        MethodInterceptor interceptor = CGLIB$CALLBACK_0;
        if (interceptor == null) {
            return super.getUser(var1);
        }
        return (String) interceptor.intercept(this, /* getUser方法引用 */, new Object[]{var1}, /* MethodProxy */);
    }

    // ... 其他被重写的方法和CGLIB的内部方法 ...
}
```

当你调用 `UserServiceImpl`

EnhancerBySpringCGLIB

`a1b2c3d4.addUser("test")` 时，会执行到重写的 `addUser` 方法。这个方法会调用 `DynamicAdvisedInterceptor` 实例的 `intercept` 方法，该方法会执行AOP通知链，并最终通过 `MethodProxy` 调用原始 `UserServiceImpl`（即父类）的 `addUser` 方法。

### 总结

- **代理生成时机**：主要在Bean初始化完成后的 `BeanPostProcessor.postProcessAfterInitialization` 阶段，由 `AbstractAutoProxyCreator` 负责。

- **源码关键**：`getBean` -> `initializeBean` -> `applyBeanPostProcessorsAfterInitialization` -> `AbstractAutoProxyCreator.wrapIfNecessary` -> `ProxyFactory.getProxy`。

- **代理类形态**：

  - **JDK动态代理**：基于接口，生成实现接口的 `$ProxyX` 类，通过 `InvocationHandler` (如`JdkDynamicAopProxy`) 转发调用。

  - **CGLIB代理**：基于继承，生成目标类的子类 `Target`

    EnhancerBySpringCGLIB

    `xxxx`，通过 `MethodInterceptor` (如`DynamicAdvisedInterceptor`) 拦截方法调用。

理解这些机制有助于你更深入地排查Spring AOP相关的问题，以及更好地利用Spring的声明式服务。