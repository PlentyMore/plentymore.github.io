---
title: ClassLoader资源加载机制
date: 2019-01-12 21:38:27
tags:
    - JavaBasic
---

Java的类加载器（[ClassLoader](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassLoader.html)）不仅可以加载类，还可以加载资源（文本文件，图片，视频等各种文件资源）。

## 三个重要的类加载器
在Java中，一般有三个类加载器， __Bootstrap Class Loader__，__ExtClassLoader__ 和 __AppClassLoader__。


### Bootstrap Class Loader

__Bootstrap Class Loader__ 称为根类加载器，它是JVM实现的，使用C++语言编写，虚拟机启动的时候会使用Bootstrap Class Loader来加载Java核心类库，比如下面的图里面的类库（比如rt.jar）：

![Imgur](https://i.imgur.com/ai6MuEE.png)

需要注意的是并不是图里面的所有类库都是根加载器加载的，里面有部分类库是拓展类加载器（ExtClassLoader）加载的。具体加载过程可以查看[JVM规范](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.3)

### ExtClassLoader

__ExtClassLoader__（拓展类加载器） 是sun.misc包下的`Launcher`类的一个静态内部类，它是用来加载ext目录下的类库的（这些类库成为拓展类库）。
```java
    /*
     * The class loader used for loading installed extensions.
     */
    static class ExtClassLoader extends URLClassLoader {
    	// 只贴出getExtDirs方法
    	// 在这个方法里面可以看到它或获取java.ext.dirs这个系统变量
        private static File[] getExtDirs() {
            String s = System.getProperty("java.ext.dirs");
            File[] dirs;
            if (s != null) {
                StringTokenizer st =
                    new StringTokenizer(s, File.pathSeparator);
                int count = st.countTokens();
                dirs = new File[count];
                for (int i = 0; i < count; i++) {
                    dirs[i] = new File(st.nextToken());
                }
            } else {
                dirs = new File[0];
            }
            return dirs;
        }
    }
```

那么ext目录是哪个目录呢？从`ExtClassLoader`里面的getExtDirs方法可以看到，ext目录是从 __java.ext.dirs__ 这个个系统变量获取的，执行下面的方法，可以得到这个系统变量的默认值
```java
public class App {

    public static void main(String[] args) {
        String s = System.getProperty("java.ext.dirs");
        System.out.println(s);
    }
}
```
这个系统变量的值默认如下图：

![Imgur](https://i.imgur.com/IOBnAKN.png)

可以看到一共有两个路径，`/opt/java/jdk1.8.0_192/jre/lib/ext`和`/usr/java/packages/lib/ext`
在我的机器上面，后面的路径是不存在的，前面的路径是存在的。

![Imgur](https://i.imgur.com/aZzbWnf.png)

上面的图中的所有类库都是由拓展类加载器（ExtClassLoader）加载的（分别有cldrdata.jar，jfxrt.jar，nashorn.jar，sunpkcs11.jar，dnsns.jar，localedata.jar，sunec.jar，zipfs.jar，jaccess.jar，meta-index，sunjce_provider.jar）


### AppClassLoader

__AppClassLoader__（系统类加载器，也就SystemClassLoader）也是sun.misc包下的`Launcher`类的一个静态内部类，它是用来加载classpath里面的类库的。

```java
    /**
     * The class loader used for loading from java.class.path.
     * runs in a restricted security context.
     */
    static class AppClassLoader extends URLClassLoader {
    	        public static ClassLoader getAppClassLoader(final ClassLoader extcl)
            throws IOException
        {
        	// 可以看到它将获取java.class.path这个系统变量，也就是classpath的值
            final String s = System.getProperty("java.class.path");
            final File[] path = (s == null) ? new File[0] : getClassPath(s);
            // ...
        }
    }
```

__AppClassLoader__ 是用来加载我们自己的类和我们依赖的其它第三方类库的，我们的类和依赖的类库都在classpath中（需要在启动jvm的时候通过`-classpath`参数指定）。

那么classpath的值是什么？在`AppClassLoader`类的getAppClassLoader方法里面，可以看到classpath实际上就是系统变量java.class.path的值。运行下面的方法：
```java
public class App {

    public static void main(String[] args) {
        final String s = System.getProperty("java.class.path");
        for(String ss:s.split(":")){
            System.out.println(ss);
        }
    }
}
```

结果如下图

![Imgur](https://i.imgur.com/jRFFhqU.png)

~~可以看到，第三方依赖包也属于classpath，我自己的工程目录也属于classpath，还有java核心类库的目录和ext目录也属于classpath~~ 实际上，使用IDE运行的时候它自动通过`-classpath`参数把图上面的jar添加到classpath（所以不要过分依赖IDE），当运行某个类的main方法的时候，默认只会把当前目录添加到classpath，如果使用了其它的第三方类库，需要使用`-classpath`指定，不然会提示无法找到依赖的类。所以，classpath的默认值如下图：

![Imgur](https://i.imgur.com/ldEPg6w.png)

只有一个目录`.`，表示当前目录


## 类加载器的关系

Bootstrap Class Loader是最顶层的类加载器，它是ExtClassLoader的父类加载器，而ExtClassLoader是AppClassLoader的父类加载器。

在sun.misc包下的`Launcher`类和java.lang包下的`ClassLoader`类里面可以看到它们直接的关系。

`ClassLoader`的getSystemClassLoader方法
```java
    public static ClassLoader getSystemClassLoader() {
        initSystemClassLoader(); // 调用了initSystemClassLoader方法
        if (scl == null) {
            return null;
        }
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkClassLoaderPermission(scl, Reflection.getCallerClass());
        }
        return scl;
    }

    private static synchronized void initSystemClassLoader() {
        if (!sclSet) {
            if (scl != null)
                throw new IllegalStateException("recursive invocation");
            // 创建Launcher，使用它获取系统类加载器
            sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
            if (l != null) {
                Throwable oops = null;
                scl = l.getClassLoader();
                try {
                    scl = AccessController.doPrivileged(
                        new SystemClassLoaderAction(scl));
                } catch (PrivilegedActionException pae) {
                    oops = pae.getCause();
                    if (oops instanceof InvocationTargetException) {
                        oops = oops.getCause();
                    }
                }
                if (oops != null) {
                    if (oops instanceof Error) {
                        throw (Error) oops;
                    } else {
                        // wrap the exception
                        throw new Error(oops);
                    }
                }
            }
            sclSet = true;
        }
    }
```

在调用`ClassLoader`的getSystemClassLoader获取系统类加载器的时候，会创建一个`sun.misc.Launcher`实例，然后用这个实例获取系统类加载器。下面来看系统类加载器是怎么创建的

```java
public class Launcher {
    public Launcher() {
        // Create the extension class loader
        ClassLoader extcl;
        try {
        	// 这里创建了ExtClassLoader的实例
            extcl = ExtClassLoader.getExtClassLoader();
        } catch (IOException e) {
            throw new InternalError(
                "Could not create extension class loader", e);
        }

        // Now create the class loader to use to launch the application
        try {
        	// 这里创建了AppClassLoader的实例，
        	// 同时将AppClassLoader父加器设为ExtClassLoader
            loader = AppClassLoader.getAppClassLoader(extcl);
        } catch (IOException e) {
            throw new InternalError(
                "Could not create application class loader", e);
        }

        // Also set the context class loader for the primordial thread.
        Thread.currentThread().setContextClassLoader(loader);

        // Finally, install a security manager if requested
        // ...
    }
}
```

可以看到在`sun.misc.Launcher`的无参构造器里面，创建了`ExtClassLoader`和`AppClassLoader`的实例，创建`AppClassLoader`的时候，使用的是`AppClassLoader`的getAppClassLoader方法，该方法最终会返回`AppClassLoader`的实例，所以调用ClassLoader.getSystemClassLoader得到的系统类加载器就是`AppClassLoader`了
```java
        public static ClassLoader getAppClassLoader(final ClassLoader extcl)
            throws IOException
        {
            final String s = System.getProperty("java.class.path");
            final File[] path = (s == null) ? new File[0] : getClassPath(s);

            // Note: on bugid 4256530
            // Prior implementations of this doPrivileged() block supplied
            // a rather restrictive ACC via a call to the private method
            // AppClassLoader.getContext(). This proved overly restrictive
            // when loading  classes. Specifically it prevent
            // accessClassInPackage.sun.* grants from being honored.
            //
            return AccessController.doPrivileged(
                new PrivilegedAction<AppClassLoader>() {
                    public AppClassLoader run() {
                    URL[] urls =
                        (s == null) ? new URL[0] : pathToURLs(path);
                    // 这里会把ExtClassLoader设置成AppClassLoader的父类加载器
                    return new AppClassLoader(urls, extcl);
                }
            });
        }

```
可以看到最后调用了`AppClassLoader`的构造方法，最终会将设置`ExtClassLoader`设置成它的父类加载器。

所以`AppClassLoader`的父类加载器为`ExtClassLoader`。那么`ExtClassLoader`的父类加载器为什么是Bootstrap Class Loader？来看到`ExtClassLoader`的构造方法：

```java
        public ExtClassLoader(File[] dirs) throws IOException {
            // ExtClassLoader的父加器将设置为null
            super(getExtURLs(dirs), null, factory);
            SharedSecrets.getJavaNetAccess().
                getURLClassPath(this).initLookupCache(this);
        }
```
可以看到`ExtClassLoader`的父类加载器将被设置为`null`，为什么设置为`null`就说它的父类加载器是Bootstrap Class Loader了？这是由Java的类加载机制————双亲委派机决定的。

## 双亲委派机制

双亲委派机制，就是当要使用类加载器加载某个类的时候，先委托该类加载器的父类加载器加载这个类，如果这个父类加载器还有父类加载器（该类加载器的parent变量不为`null`），则继续委托它的父类加载器加载这个类，直到父类加载器的parent变量为`null`的时候，就使用findBootstrapClass方法找到Bootstrap Class Loader，然后使用Bootstrap Class Loader加载类，如果加载成功了，则返回，如果加载失败了，则让刚刚委派它的父类加载器来加载类，如果也加载失败了，则让委派这个父类加载器加载的类加载器来加载类，然后如果一直加载失败，最后会回到一开始使用的类加载器，然后使用这个类加器加载类。具体过程如下：
```java
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

所以，当一个加载器的parent变量为`null`的时候，就认为它的父类加载器为Bootstrap Class Loader，而Bootstrap Class Loader是最顶层的父类加载器，它已经没有父类加载器了，所以不会再委派其父类加载器加载类。

需要注意的是loadClass方法不是final的，所以是可以被子类重写的，所以理论上我们也可以不使用双亲委派机制来加载类。

所以，为什么要让父类加载器先加载呢？假设不让父类加载器先加载类（即不按照双亲委派机制来进行加载），JVM启动的时候，会使用Bootstrap Class Loader加载核心的类库，比如rt.jar，然后使用ExtClassLoader加载拓展类库，然后使用AppClassLoader来加载classpath中的类库，如果没有双亲委派机制，那么AppClassLoader可能会把Bootstrap Class Loader和ExtClassLoader加载过的类又重新加载一遍（假设Bootstrap Class Loader和ExtClassLoader加载的类库都被添加到了classpath中），使得同一个类被不同的类加载器多次加载，这样的话会造成同一个类有不同的类加载器。如果让父类加载器先加载，那么rt.jar里面的类只会被Bootstrap Class Loader加载，拓展类库里面的类只会被ExtClassLoader加载（因为它的父类加载器为Bootstrap Class Loader，而Bootstrap Class Loader只能加载核心类库），这样就保证了JVM中的同一个类不会有不同的类加载器。


## 资源加载机制

`ClassLoader`的资源加载机制和类加载机制一样，都遵循双亲委派机制。

### ClassLoader的getResource方法

下面看到它的getResource方法
```java
    public URL getResource(String name) {
        URL url;
        if (parent != null) {
        	// 使用父类加载器的的getResource加载
            url = parent.getResource(name);
        } else {
        	// parent为null，所没父加器为根加载器，使用根加载器进行加载
            url = getBootstrapResource(name);
        }
        if (url == null) {
        	// 父类加载器没有加载到资源，则自己加载
            url = findResource(name);
        }
        return url;
    }
```

`ClassLoader`的getResource方法和类加载一样，遵循双亲委派机制。`ClassLoader`的getSystemResource方法和`Class`的getResource方法实际上最终都会调用`ClassLoader`的getResource方法按照双亲委派机制加载资源（如果类加载器重写了getResource方法，则不一定会按照该机制加载资源），如果父类加载器没有成功加载到资源，则会调用自己的findResource方法加载资源。

`URLClassLoader`实现了findResource方法，而ExtClassLoader和AppClassLoader都继承了`URLClassLoader`，并且都没有重写这个方法，所以调用它们的findResource方法的时候实际上是调用`URLClassLoader`实现的findResource方法。

```java
    public URL findResource(final String name) {
        /*
         * The same restriction to finding classes applies to resources
         */
        URL url = AccessController.doPrivileged(
            new PrivilegedAction<URL>() {
                public URL run() {
                    return ucp.findResource(name, true);
                }
            }, acc);

        return url != null ? ucp.checkURL(url) : null;
    }
```
可以看到是调用了ucp来加载资源，ucp是`sun.misc.URLClassPath`类型的

```java
public class URLClassPath {
    public URL findResource(String name, boolean check) {
        Loader loader;
        int[] cache = getLookupCache(name);
        for (int i = 0; (loader = getNextLoader(cache, i)) != null; i++) {
        	// 调用了loader的findResource
            URL url = loader.findResource(name, check);
            if (url != null) {
                return url;
            }
        }
        return null;
    }
}
```

然而`sun.misc.URLClassPath`实际上是使用`Loader`加载资源的，`Loader`是它的一个内部类，它是用来加载网络资源的，`Loader`还有两个子类，分别为`FileLoader`（用来加载本地资源）和`JarLoader`（用来加载jar里面的资源）
```java
    private Loader getLoader(final URL url) throws IOException {
        try {
            return java.security.AccessController.doPrivileged(
                new java.security.PrivilegedExceptionAction<Loader>() {
                public Loader run() throws IOException {
                    String file = url.getFile();
                    if (file != null && file.endsWith("/")) {
                        if ("file".equals(url.getProtocol())) {
                        	// url以file开头，职责创建FileLoader
                            return new FileLoader(url);
                        } else {
                        	// 不以file开头说明是网络资源，创建Loader
                            return new Loader(url);
                        }
                    } else {
                    	// 是jar资源，创建JarLoader
                        return new JarLoader(url, jarHandler, lmap);
                    }
                }
            });
        } catch (java.security.PrivilegedActionException pae) {
            throw (IOException)pae.getException();
        }
    }
```


### ClassLoader的getSystemResource方法

```java
    public static URL getSystemResource(String name) {
        // 获取SystemClassLoader
        ClassLoader system = getSystemClassLoader();
        if (system == null) {
        	// 系统SystemClassLoader为空，则使用根类加载器
            return getBootstrapResource(name);
        }
        // 使用类加载器加载资源
        return system.getResource(name);
    }
```
getSystemResource方法会调用getSystemClassLoader方法获取SystemClassLoader（系统类加载器），SystemClassLoader就是`AppClassLoader`，下面查看getSystemClassLoader进行验证：
```java
    public static ClassLoader getSystemClassLoader() {
        // 这里将初始化SystemClassLoader
        initSystemClassLoader();
        if (scl == null) {
            return null;
        }
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkClassLoaderPermission(scl, Reflection.getCallerClass());
        }
        return scl;
    }
```
getSystemClassLoader方法调用了initSystemClassLoader方法来初始化SystemClassLoader：
```java
    private static synchronized void initSystemClassLoader() {
        if (!sclSet) {
            if (scl != null)
                throw new IllegalStateException("recursive invocation");
            // 创建了Launcher实例
            sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
            if (l != null) {
                Throwable oops = null;
                // 调用那个Launcher的getClassLoader方法
                // 该方法会返回Launcher的loader变量
                // 而这个变量的值就是AppClassLoader实例
                scl = l.getClassLoader();
                try {
                	// 这里还会调用SystemClassLoaderAction
                	// 它会获取系统变量的值，使用该值创建SystemClassLoader
                    scl = AccessController.doPrivileged(
                        new SystemClassLoaderAction(scl));
                } catch (PrivilegedActionException pae) {
                    oops = pae.getCause();
                    if (oops instanceof InvocationTargetException) {
                        oops = oops.getCause();
                    }
                }
                // ...
            }
            sclSet = true;
        }
    }
```

首先它会创建一个`sun.misc.Launcher`实例，然后用该实例的getLauncher方法获取SystemClassLoader，实际上getLauncher会返回`sun.misc.Lanucher`的loader变量，而该变量是在`sun.misc.Lanucher`的构造器里面这这样初始化的：
```java
loader = AppClassLoader.getAppClassLoader(extcl);
```
所以返回的loader为`AppClassLoader`，然而这还没有结束，后面还会调用`SystemClassLoaderAction`的run方法来获取显式指定的SystemClassLoader：
```java
class SystemClassLoaderAction
    implements PrivilegedExceptionAction<ClassLoader> {
    private ClassLoader parent;

    SystemClassLoaderAction(ClassLoader parent) {
        this.parent = parent;
    }
    public ClassLoader run() throws Exception {
    	// 获取java.system.class.loader系统变量
    	// 如果为空，则直接返回parent，即AppClassLoader
    	// 否则使用该值创建ClassLoader，并作为SystemClassLoader返回
        String cls = System.getProperty("java.system.class.loader");
        if (cls == null) {
            return parent;
        }

        Constructor<?> ctor = Class.forName(cls, true, parent)
            .getDeclaredConstructor(new Class<?>[] { ClassLoader.class });
        ClassLoader sys = (ClassLoader) ctor.newInstance(
            new Object[] { parent });
        Thread.currentThread().setContextClassLoader(sys);
        return sys;
    }
}
```

首先`SystemClassLoaderAction`的run方法会获取系统变量java.system.class.loader的值，这个值如果不在jvm参数里面指定的话是默认为空的。如果该值为空，则直接返回parent变量，也就是`AppClassLoader`变量，所以默认情况下，SystemClassLoader（系统类加载器）为`AppClassLoader`，如果在jvm参数里面指定了java.system.class.loader，则会使用java.system.class.loader指定的类作为SystemClassLoader，也就是说SystemClassLoader是可以改变的，不一定为`AppClassLoader`。


现在我们已经知道了getSystemClassLoader方法默认会返回`AppClassLoader`了，回到ClassLoader的getSystemResource方法，因为现在已经获取到了SystemClassLoader（类型为`AppClassLoader`），所以将调用这个SystemClassLoader的getResource方法加载资源，也就是`AppClassLoader`的getResource方法。

实际上`AppClassLoader`没有getResource方法，它的getResource方法是从`ClassLoader`继承的，上面已经提到了`ClassLoader`的getResource方法，这里就不再复读了。

所以，在默认情况下，ClassLoader的getSystemResource方法将使用`AppClassLoader`来加载资源。


### Class的getResource方法

在`Class`中也有一个getResource方法
```java
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
    public java.net.URL getResource(String name) {
        // 调用resolveName解析资源名称
        // 如果name是以/开头，则会去掉/然后返回
        // 如果不是一/开头，则在name前面加上该类的包名
        name = resolveName(name);
        // 这里将获取加载该类的加载器
        ClassLoader cl = getClassLoader0();
        if (cl==null) {
            // A system class.
            // 如果获取到的类加载器为空，则使用系统类加载器来加载资源
            return ClassLoader.getSystemResource(name);
        }
        return cl.getResource(name);
    }
}
```
这个方法在加载资源之前，会先调用resolveName方法解析资源名称：
```java
    private String resolveName(String name) {
        if (name == null) {
            return name;
        }
        // name不是以/开头，表示从包名的路径开始查找
        if (!name.startsWith("/")) {
            Class<?> c = this;
            while (c.isArray()) {
                c = c.getComponentType();
            }
            // 获取类的全限定名
            String baseName = c.getName();
            // 后面将获取类所在的包的包名
            int index = baseName.lastIndexOf('.');
            if (index != -1) {
            	// 将包名的.符号替换成/，也就是变成路径的形式
                name = baseName.substring(0, index).replace('.', '/')
                    +"/"+name;
            }
        } else {
        	// 如果是/开头，则说明是从classpath开始查找，去掉/，保留后面的名称
            name = name.substring(1);
        }
        return name;
    }
```
可以这个方法会判断name是不是以`/`开头，如果是，则说明是从classpath加载相应的资源，如果不是，则说明是从当前的类所在的包的路径开始查找。下面有一个例子：

假设有一个类，全限定名称为`com.pltm.App`
```java
package com.pltm;
public class App{
	public static void main(String[] args){
		Class<App> clazz = App.class;
		clazz.getResource("a.txt");
		clazz.getResource("/a.txt");
	}
}
```
调用clazz.getResource("a.txt")的时候，将会先调用resolveName解析name，因为"a.txt"不是以`/`开头，所以最后的name将变成`com/pltm/a.txt`，所以相当于从包名开始解析资源
调用clazz.getResource("/a.txt")的时候，也是先调用resolveName解析name，因为"/a.txt"是以`/`开头的，所以最后的name将变成`a.txt`，所以相当于从classpath解析资源

回到`Class`的getResource方法，在解析完name之后，将调用getClassLoader0方法获取加载这个类的类加载器：
```java
    // Package-private to allow ClassLoader access
    ClassLoader getClassLoader0() { return classLoader; }

    // Initialized in JVM not by private constructor
    // This field is filtered from reflection access, i.e. getDeclaredField
    // will throw NoSuchFieldException
    private final ClassLoader classLoader;
```
getClassLoader0返回的是classLoader变量，这个变量是通过`Class`的私有构造器初始化的：
```java
    /*
     * Private constructor. Only the Java Virtual Machine creates Class objects.
     * This constructor is not used and prevents the default constructor being
     * generated.
     */
    private Class(ClassLoader loader) {
        // Initialize final field for classLoader.  The initialization value of non-null
        // prevents future JIT optimizations from assuming this final field is null.
        classLoader = loader;
    }
```
从注释上了解到，只有JVM才能创建`Class`对象，所以这个ClassLoader到底是什么？我们可以通过设置断点的方式查看这个ClassLoader的值，在`Class`的getResource(String name)方法中的`ClassLoader cl = getClassLoader0();`这行代码设置一个断点，然后运行下面的代码：
```java
package com.pltm;
public class App{
	public static void main(String[] args){
		Class<App> clazz = App.class;
		clazz.getResource("a.txt");
	}
}
```
结果如下图：

![Imgur](https://i.imgur.com/S9vqzPQ.png)

是`Launcher$AppClassLoader`，也就是默认的SystemClassLoader（系统类加载器），它是用来加载classpath中的类的，所以上面例子中的`App`类是又它加载的，所以可以猜测getClassLoader0方法返回的ClassLoader为负责加载当前的`Class`的ClassLoader。

这里获取到了ClassLoader，后面将会使用这个ClassLoader加载资源，也就是使用`AppClassLoader`加载资源，这不就和ClassLoader的getSystemResource方法一样了吗？是的，如果`Class`是由`AppClassLoader`加载的，并且默认的SystemClassLoader为`AppClassLoader`，那么除了在加载资源前会调用resolveName解析资源名之外，`Class`的getResource方法和ClassLoader的getSystemResource方法几乎是一样的。


## 测试

运行下面的程序，会输出什么（不打包成jar）？
```java
package com.pltm.rt;
public class App{
	public static void main(String[] args){
        Class<App> clazz = App.class;
        ClassLoader loader = clazz.getClassLoader();
        System.out.println(clazz.getResource("/"));
        System.out.println(loader.getResource("/"));
        System.out.println(ClassLoader.getSystemResource("/"));
        System.out.println(clazz.getResource(""));
        System.out.println(loader.getResource(""));
        System.out.println(ClassLoader.getSystemResource(""));
    }
}
```
程序的目录结构如下图

![Imgur](https://i.imgur.com/2hdxmOa.png)

程序输出如下图

![Imgur](https://i.imgur.com/ZZM2Xgp.png)


我是在 __/home/plentymore/projects/javabasic/target/classes__ 目录运行程序的，所以这个目录会被添加到classpath。

输出结果为
```bash
file:/home/plentymore/projects/javabasic/target/classes/
null
null
file:/home/plentymore/projects/javabasic/target/classes/com/pltm/rt/
file:/home/plentymore/projects/javabasic/target/classes/
file:/home/plentymore/projects/javabasic/target/classes/

```
接下来分析为什么会是这样的结果：
clazz.getResource("/")将调用`Class`的方法解析资源名称，因为资源名称是`/`开头的，所以`name = name.substring(1);`，所以最终的name为空字符串`”“`，接着使用加载`App`类的类加载器，即系统类加载器（实际类型为`AppClassLoader`）来加载资源，loader变量为`AppClassLoader`，ClassLoader.getSystemResource也是使用`AppClassLoader`来加载资源的，所以clazz.getResource("/")也就等同于loader.getResource("")和ClassLoader.getSystemResource("")，所以这三个的输出结果都是`file:/home/plentymore/projects/javabasic/target/classes/`。

然后看到loader.getResource("/")和ClassLoader.getSystemResource("/")，它们都不会解析name，所以，将会在classpath中查找`/`这个资源，然而这个资源不存在于classpath，所以返回`null`。

最后看到clazz.getResource("")，它先会调用`Class`的resolveName方法解析资源名称，因为资源名称不是以`/`开头的，所以会被加上类所在的包名，即`name = "com/pltm/rt" + "/" + name`，最终name的值为`com/pltm/rt/`，它是一个目录，而且这个目录是存在的，所以最终的结果为`file:/home/plentymore/projects/javabasic/target/classes/com/pltm/rt/`


如果把程序打包成jar再运行，得到的输出结果和上面的也不一样，打包成jar后运行的结果如下：

![Imgur](https://i.imgur.com/0cKapRI.png)

输出结果为
```bash
null
null
null
jar:file:/home/plentymore/projects/javabasic/target/java-basic-1.0.jar!/com/pltm/rt/
null
null

```
可以看到只有clazz.getResource("")这个调用结果不为空，为什么会变成这样呢？

实际上是因为classpath的值不一样了。

当没有打包成jar的时候，会把当前路径`.`（也就是`/home/plentymore/projects/javabasic/target/classes/`）这个路径添加到classpath。而在打包成jar之后，会把jar所在的路径`java-basic-1.0.jar`（也就是`/home/plentymore/projects/javabasic/target/java-basic-1.0.jar`）加入classpath，区别就是，一个是路径，一个是jar文件，为路径的时候，会创建FileLoader加载资源，为jar的时候，会创建JarLoader加载资源，JarLoader只能加载JarLoader里面的资源。

clazz.getResource("/")和loader.getResource("")还有ClassLoader.getSystemResource("")含义是相同的，即它们都是查找classpath中的`""`资源，但是`JarLoader`对于`""`的解析和`FileLoader`不一样，`JarLoader`遇到查找的资源名称为`""`的时候，最终会返回`null`，而`FileLoader`则是返回路径的目录对应的`URL`，所以打包成jar后这三个方法都返回`null`。

loader.getResource("/")和ClassLoader.getSystemResource("/")的含义也是一样的，都是在classpath中查找名称为`/`的资源，这个资源是不存在的，所以返回`null`。

最后看clazz.getResource("")，它会首先调用`Class`的resolveName方法解析资源名称，最终名称变成`com/pltm/rt/`，所以它实际上会在classpath中查找名称为`com/pltm/rt/`资源，因为jar文件是有这个目录的，所以就返回这个目录的路径对应的`URL`，所以结果为`jar:file:/home/plentymore/projects/javabasic/target/java-basic-1.0.jar!/com/pltm/rt/`


## 总结

在Spring的项目中，经常能看到某个方法或者构造器需要接收`ClassLoader`，其中一个很重要的作用就是使用接收的`ClassLoader`加载资源，如果传入的`ClassLoader`为空，就会使用SystemClassLoader（系统类加载器）来加载资源，该加载器能加载classpath里面的资源。还要注意的是`Class`的getResource方法，这个方法在加载资源之前会对资源名称进行解析，如果资源名称以`/`开头，则去掉资源名称的`/`，然后从classpath中查找相应的资源，如果资源名称不以`/`开头，则说明要在类对应的包下查找资源，资源的名称会加上包名对应的目录名，然后在classpath中查找该资源。