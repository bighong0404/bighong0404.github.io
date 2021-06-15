##### 1. 加载mybatis配置文件
>MyBatis提供的Resources类加载mybatis的配置文件
            `Reader reader = Resources.getResourceAsReader("mybatis.cfg.xml");`

Resources.getResourceAsReader()还有几个重载方法, 区分环境用的, 这里先忽略
`Resources`类是通过ClassLoader去访问资源文件的. 内部有个成员变量`ClassLoaderWrapper classLoaderWrapper`, 调用`ClassLoaderWrapper`的`getClassLoaders ()`方法获取classLoader数组. 
```
ClassLoader[] getClassLoaders(ClassLoader classLoader) {
    return new ClassLoader[]{
        classLoader,
        defaultClassLoader,
        Thread.currentThread().getContextClassLoader(),
        getClass().getClassLoader(),
        systemClassLoader};
  }
```
任意一个classLoader读取到资源, 即完成操作.
```
InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
    for (ClassLoader cl : classLoader) {
      if (null != cl) {

        // try to find the resource as passed
        InputStream returnValue = cl.getResourceAsStream(resource);

        // now, some class loaders want this leading "/", so we'll add it and try again if we didn't find the resource
        if (null == returnValue) {
          returnValue = cl.getResourceAsStream("/" + resource);
        }

        if (null != returnValue) {
          return returnValue;
        }
      }
    }
    return null;
  }
```










##### 2. 构建sessionFactory
