# 1. 类加载过程
在Java中，类装载器把一个类装入Java虚拟机中，主要经过三个步骤来完成：**加载、连接和初始化**，其中链接又可以分成校验、准备和解析三步. 

除了解析外，其它步骤是严格按照顺序完成的，各个步骤的主要工作如下：

- 加载：查找和导入类或接口的二进制数据； 
- 连接：又可以分成校验、准备和解析三步，其中解析步骤是可以选择的； 
  - 校验：检查导入类或接口的二进制数据的正确性； 
  - 准备：给类静态变量分配内存, 设置类变量初始值； 
  - 解析：将常量池内的符号引用替换为直接引用； 
- 初始化：初始化Java代码的静态变量, 以及执行静态代码块。


---

# 2. ClassLoader.loadClass()与Class.forName()的源码
```
// ClassLoader.Java
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {

        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            //...
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

    /**
     * Links the specified class.  执行链接
     */
    protected final void resolveClass(Class<?> c) {
        resolveClass0(c);
    }
    // native方法
    private native void resolveClass0(Class<?> c);
```
- 从以上`ClassLoader.loadClass()`的源码来看, 由于并不会执行`resolveClass(Class<?> c)`**链接**, 因此也不会执行加载的第三步骤**初始化**. 
- 而`Class.forName()`是会执行**初始化**. 

```
//CLass.java
    @CallerSensitive
    public static Class<?> forName(String className)
                throws ClassNotFoundException {
        Class<?> caller = Reflection.getCallerClass();
        return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    }

    /** Called after security check for system loader access checks have been made. */
    private static native Class<?> forName0(String name, boolean initialize,
                                            ClassLoader loader,
                                            Class<?> caller)
        throws ClassNotFoundException;
```
