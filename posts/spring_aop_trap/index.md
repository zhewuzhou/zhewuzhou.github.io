# Spring AOP：内部调用陷阱


## 摘要
最近写代码，遇到一个 奇怪的Spring AOP 有关的问题；本文从这个问题出发，通过问问题的方式揭示这个问题背后深层原因。

## AOP 问题代码清单
- AOP Aspect: HttpHeaderValidator.java
``` java
@Aspect
public class HttpHeaderValidator {
	@Before
	public void isUserInfoExist(){
	  ...
	}
}
```
- AOP Joint Point: Logger.java
``` java
@Aspect
public class Logger {
	public void logTransactionA(){
      String user = this.getUserFromHeader();
	  ...
	}
	
	public void logTransactionB(){
	  String user = this.getUserFromHeader();
	  ...
	}
	
	@UserValidator
	public String getUserFromHeader(){
	  ...
	}
}
```
代码的逻辑是，通过 UserValidator 将检查 Http Header 用户信息的切片逻辑插入到方法 getUserFromHeader() 之前；由于 Logger 类多处写日志的方法都会调用 getUserFromHeader() 因此，也就等价于多处写日志的时候都会进行 Header 的检查，避免了在每一个写日志的方法上加上 annotation.

然而执行代码却发现切片逻辑根本没有被执行；如果换种写法，把 UserValidator 加到每一个写日志的方法上，切片逻辑被调用了；这是为什么呢？

## 表面原因
Spring AOP 文档中有如下描述：
> Okay, so what is to be done about this? The best approach (the term best is used loosely here) is to refactor your code such that the self-invocation does not happen. For sure, this does entail some work on your part, but it is the best, least-invasive approach. The next approach is absolutely horrendous, and I am almost reticent to point it out precisely because it is so horrendous. 

上述代码切面逻辑之所以没有被调用，原因在于方法 getUserFromHeader 的调用发生在类 Logger 内部 (self-invocation) , 因此上述代码是不工作的，[Spring官方文档][1] 还给出了一段解释：
> The key thing to understand here is that the client code inside the main(..) of the Main class has a reference to the proxy. This means that method calls on that object reference will be calls on the proxy, and as such the proxy will be able to delegate to all of the interceptors (advice) that are relevant to that particular method call. However, once the call has finally reached the target object, the SimplePojo reference in this case, any method calls that it may make on itself, such as this.bar() or this.foo(), are going to be invoked against the this reference, and not the proxy. This has important implications. It means that self-invocation is not going to result in the advice associated with a method invocation getting a chance to execute.

按照这个解释，在  logTransactionA() 方法被调用的时候，其内部调用 getUserFromHeader() 指向的是 this 也就是 Logger 类对象，而不是调用前插入切片逻辑的的代理方法。

只要注意写 AOP 代码的时候不要出现内部调用代理方法，似乎这个问题就得到了解决。然而更多的问题浮现出来：
- **为什么内部调用会导致 Spring AOP 失效?**
- **为什么内部调用的时候调用指向 this 指针而非 proxy 方法？**

## 进一步思考

内部调用导致 Spring AOP 失效的问题本质实际上是一个设计问题；要弄清楚这个问题，需要在更高的层次上了解更多的细节。

###  AOP 的几种编织方式

在进一步探索之前有必要了解下 AOP 的几种编织方式；所谓编织即切片逻辑插入切入点的过程；有编译时编织、加载时编织和运行时编织等多种方式，Spring AOP 的编织方式是运行时编织；尽管编译时编织提供更好的灵活性，比如甚至可以将切片逻辑插入到某一具体代码行附近，然而相比于运行时编织其依赖更多。

运行时编织：即运行时基于 Java Dynamic Proxy 特性（基于接口），或者基于 CGLib、ByteBuddy 等（基于实现类），通过子类 (Proxy) 的方式将切片逻辑和切入点函数调用粘连到一起；Spring AOP 实际上是运行时编织，其编织粒度是函数运行级。
![运行时编织](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/SpringAOP.png)

### 新问题

既然 Spring AOP 通过 Proxy 也就是 Subclass 的方式实现，那么其他类调用方法 logTransactionA() 时候实际上通过代理，而代理除了切片逻辑之外，肯定需要调用父类 Logger 的 logTransactionA() 方法，假设其调用方式为 super.logTransactionA()，那么由于多态的存在，最终 代理类中 getUserFromHeader() 应该被调用，也就是即便是内部调用，切片逻辑也应该被调用到；既然 AOP 在内部调用场景下失效，那么代理类在调用父类的方法的时候，必然不是简单的 super.logTransactionA()，**那么代理类究竟是如何调用父类的方法的呢？**

这个问题可以再源码中找到答案，首先 Spring AOP 的某人代理创建工厂如下：
``` java
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
基于 CGlib 的 ObjenesisCglibAopProxy 会创建 Proxy 对象，然后通过 CGLib 回调的方式，切片逻辑被插入到切入点。限于篇幅，有关于 CGLib Callback 的细节请参考  [CGLib Callback 细节][2]

本文所举的 Spring AOP 场景下，最终回调类 DynamicAdvisedInterceptor 的 intercept 方法会被调用，这个方法如下：
``` java
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// Check whether we only have one InvokerInterceptor: that is,
				// no real advice, but just reflective invocation of the target.
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					// We can skip creating a MethodInvocation: just invoke the target directly.
					// Note that the final invoker must be an InvokerInterceptor, so we know
					// it does nothing but a reflective operation on the target, and no hot
					// swapping or fancy proxying.
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// We need to create a method invocation...
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}
```
调用 proxy 类同名方法，最终会通过 CGlib 调用到对应的回调类 intercept 方法，在调用此方法的时候具体的父类方法和参数都被具化 (Method 类)；也就是在最后的调用中一定是通过被代理类的实例完成调用，即表面上通过代理继承父类的方式，实际上子类在调用父类方法的时候通过父类的实例调用，因此在本文描述的内部调场景下，切片逻辑没有被调用。

``` java
	public Object invoke(Object obj, Object[] args) throws Throwable {
		try {
			init();
			FastClassInfo fci = fastClassInfo;
			return fci.f1.invoke(fci.i1, obj, args);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
		catch (IllegalArgumentException ex) {
			if (fastClassInfo.i1 < 0)
				throw new IllegalArgumentException("Protected method: " + sig1);
			throw ex;
		}
	}
```

上面的源码摘自 ProxyMethod ，父类的方法正是通过这个方法被调用的，这里可以看参数 obj ，这个参数正是父类的实例，而这正是多态没有生效的真正原因。

## 总结

本文从一个 Spring AOP 失效的场景出发，通过 Spring 官方文档，发现其有讲到本文所描述的内部调用场景下 AOP 会失效，然而并没有给出确切的原因。

如果考虑多态，即便是内部调用，AOP 也不该失效；通过源码和 Spring AOP 的设计思想，内部调用正真的失效原因如下：
- Spring AOP 不支持编译时编织；而编译时编织提供了最大的灵活性，支持内部调用 AOP 并不是难事。
- 最终当父类的方法被调用的时候，已经是具体化了 Method 类，因此多态不会生效。

[1]: https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-proxying
[2]: https://dzone.com/articles/cglib-missing-manual

