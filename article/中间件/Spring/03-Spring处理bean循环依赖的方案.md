> Spring对循环依赖的解决方法
>
> 这里只讨论单例setter循环依赖情况. 因为其他循环依赖spring直接报错
>
> 参考: https://www.iflym.com/index.php/code/201208280001.html



直接抛出答案: **Spring通过建立三级缓存来解决循环依赖问题**



# 三个缓存Map的说明

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

  /** Cache of singleton objects: bean name to bean instance. */
	// 一级缓存，key beanName，value就是beanName对应的单实例对象引用。
	// 二三级缓存都只是临时容器, 一旦对象创建最终完成, 存放到一级缓存中, 同时把对象从二三级缓存清除
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	// 三级缓存, 用于存储在spring内部所使用的beanName->对象工厂的引用，一旦最终对象被创建(通过objectFactory.getObject())，此引用信息将删除
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	// 二级缓存, 保存早期实例对象, 提前曝光的
	// 用于存储在创建Bean早期对创建的原始bean的一个引用(注意这里是原始bean)，即使用工厂方法或构造方法创建出来的对象，一旦对象最终创建好，此引用信息将删除
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
  
}
```

可以看出，这`singletonFactories`, `earlySingletonObjects`都是一个临时工。在所有的对象创建完毕之后，此两个缓存的size都为0。



# 通过几个关键方法来看看三个缓存的协作方式.



```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry
```

- M1 - `getSingleton()` 从缓存获取实例对象.

```java
	/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		//到一级缓存中获取beanName对应的单实例对象。
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				//检查二级缓存是否存在实例对象
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
          // 二级缓存没有, 去三级缓存找
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					//3级有数据, 缓存升级。
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						//保存到2级缓存
						this.earlySingletonObjects.put(beanName, singletonObject);
						//把工厂类对象从3级缓存中清调
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}

```

- M2 - `getSingleton()`, 通过工厂类创建实例

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				// 省略无关代码...
        
				try {
					// ★ 通过ObjectFactory获取实例对象
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				
        // 省略....
        
				if (newSingleton) {
					// ★ 创建完成后, 注册到一级缓存singletonObjects与registeredSingletons, 并且从二三级缓存中移出
          // 对应M4方法
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```



- M3 - `addSingletonFactory()`, 创建实例对象工厂类. 默认情况下, 所有单实例对象在创建时候都会交由ObjectFactory创建, 并把ObjectFactory对象保存到三级缓存中.

```java
	/**
	 * Add the given singleton factory for building the specified singleton
	 * if necessary.
	 * <p>To be called for eager registration of singletons, e.g. to be able to
	 * resolve circular references.
	 * @param beanName the name of the bean
	 * @param singletonFactory the factory for the singleton object
	 */
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				// 存到三级缓存
				this.singletonFactories.put(beanName, singletonFactory);
				// 从二级缓存删除
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

- M4 - `addSingleton`(), 实例对象最终创建完成时候调用

```java
	/**
	 * Add the given singleton object to the singleton cache of this factory.
	 * <p>To be called for eager registration of singletons.
	 * @param beanName the name of the bean
	 * @param singletonObject the singleton object
	 */
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			//实例对象最终创建完成, 保存到一级缓存
			this.singletonObjects.put(beanName, singletonObject);
			// 并从三级缓存删除
			this.singletonFactories.remove(beanName);
			// 并从二级缓存删除
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}

```

# 单实例bean的获取/创建过程



类`AbstractBeanFactory`

```java
protected <T> T doGetBean(
  String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
  throws BeansException {
  	// 省略无关代码....
  
  	//★ 到缓存中获取共享单实例。 对应上面M1方法
		Object sharedInstance = getSingleton(beanName);
  	if (sharedInstance != null && args == null) {
      // 省略....
    }else{
      
      // 省略...
      
      if (mbd.isSingleton()) {
					//★ getSingleton重载方法，这个方法更倾向于创建实例, 存放到一级缓存中, 并返回。
        	// 对应上面M2方法
					sharedInstance = getSingleton(beanName, () -> {
						try {
              // ★ 创建
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
      }
      
      // 省略....
    }
}
```

`createBean(beanName, mbd, args);`代码最终会走到`doCreateBean()`

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
  // 省略无关代码...
  
  //★ 该方法创建出来真实的bean实例，并且将其包装到BeanWrapper实例中。
  createBeanInstance(beanName, mbd, args);
  
  //早期实例是否暴露到三级缓存
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                    isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
      logger.trace("Eagerly caching bean '" + beanName +
                   "' to allow for resolving potential circular references");
    }
    // ★ 生成的匿名ObjectFactory对象存放到三级缓存中
    // 对应上面M3方法
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }
  
  // 省略属性填充, 初始化, 注册销毁等代码...
}
```



# 总结

## 文字描述单实例对象规避循环调用的创建步骤

1. `getSingleton()`-**M1**方法从缓存查询
   - 一级缓存SingletonObjects获取不到, 去二级缓存`earlySingletonObjects`找
   - 二级缓存找不到, 去三级缓存`singletonFactories`找
   - 三级缓存找到了, 通过ObjectFactory.getObject()得到实例对象, 保存到二级缓存, 并从三级缓存删除

2. 若`getSingleton()`(**M1**方法)找不到, 进行创建实例步骤: 
   - `getSingleton`(beanName, () -> { return `createBean(beanName, mbd, args)`; }) -**M2方法**
   - ``createBean()` -> `doCreateBean()` -> `addSingletonFactory()` -**M3方法**, 把objectFactory工厂对象保存到三级缓存.
   - `doCreateBean()`继续执行后面的属性填充, 初始化等生命周期流程代码.  执行结束后, 回到M2方法继续执行. 这时候bean对象已经是最终创建完成状态.
   - `getSingleton()`-M2方法通过objectFactory.getObject()获取到创建完成的实例对象后, 执行`addSingleton`-**M4方法**, 把实例对象保存到一级缓存, 并删除二三级缓存相关的对象.



## 通过图表表示步骤

> 以下有/无，表示是否持有对指定Bean的引用

在正常的情况下，调用顺序如下：

|                                         | 三级缓存singletonFactories | 二级缓存earlySingletonObjects | 一级缓存singletonObjects |
| --------------------------------------- | -------------------------- | ----------------------------- | ------------------------ |
| getSingleton(beanName, true)            | 无                         | 无                            | 无                       |
| doCreateBean(beanName,mdb,args)         | 有                         | 无                            | 无                       |
| getSingleton(beanName, true);           | 有                         | 无                            | 无                       |
| addSingleton(beanName, singletonObject) | 无                         | 无                            | 有                       |



出现循环引用之后呢，就会出现这种情况：

|                                                     | 三级缓存singletonFactories | 二级缓存earlySingletonObjects | 一级缓存singletonObjects |
| --------------------------------------------------- | -------------------------- | ----------------------------- | ------------------------ |
| getSingleton(A, true);                              | A无B无                     | A无B无                        | A无B无                   |
| doCreateBean(A,mdb,args)                            | A有B无                     | A无B无                        | A无B无                   |
|                                                     |                            |                               |                          |
| populateBean(A, mbd, instanceWrapper) 解析B……       |                            |                               |                          |
| getSingleton(B, true)                               | A有B无                     | A无B无                        | A无B无                   |
| doCreateBean(B,mdb,args)                            | A有B有                     | A无B无                        | A无B无                   |
| populateBean(B, mbd, instanceWrapper)由B准备解析A…… |                            |                               |                          |
| getSingleton(A, true)                               | A无B有                     | A有B无                        | A无B无                   |
| 完成populateBean(B, mbd, instanceWrapper)解析……     |                            |                               |                          |
| addSingleton(B, singletonObject)                    | A无B无                     | A有B无                        | A无B有                   |
| 完成populateBean(A, mbd, instanceWrapper)           |                            |                               |                          |
| addSingleton(A, singletonObject)                    | A无B无                     | A无B无                        | A有B有                   |

