---
layout:     post
title:      "spring源码分析：bean的加载流程"
subtitle:   "从知道不知道到知道知道。"
date:       2017-04-15 11:00:00
author:     "Jianxin Guo"
header-img: "img/post-bg-01.jpg"
catalog: true                       # 是否归档
tags:                               #标签
    - Spring
---


下面有很简单的一段代码可以作为Spring代码加载的入口：
 
```
1 ApplicationContext ac = new ClassPathXmlApplicationContext("spring.xml");
2 ac.getBean(XXX.class);
```
ClassPathXmlApplicationContext用于加载CLASSPATH下的Spring配置文件，可以看到，第二行就已经可以获取到Bean的实例了，那么必然第一行就已经完成了对所有Bean实例的加载，因此可以通过ClassPathXmlApplicationContext作为入口。为了后面便于代码阅读，先给出一下ClassPathXmlApplicationContext这个类的继承关系：

<a href="#">
    <img src="{{ site.baseurl }}/img/spring01-01.png" alt="Post Sample Image">
</a>

看下ClassPathXmlApplicationContext的构造函数：
 
```
1 public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
2     this(new String[] {configLocation}, true, null);
3 }
```

```
1 public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
2         throws BeansException {
3 
4     super(parent);
5     setConfigLocations(configLocations);
6     if (refresh) {
7         refresh();
8     }
9 }
```

从第二段代码看，总共就做了三件事：

1、super(parent)

　　没什么太大的作用，设置一下父级ApplicationContext，这里是null

2、setConfigLocations(configLocations)

　　代码就不贴了，一看就知道，里面做了两件事情：

　（1）将指定的Spring配置文件的路径存储到本地

　（2）解析Spring配置文件路径中的${PlaceHolder}占位符，替换为系统变量中PlaceHolder对应的Value值，System本身就自带一些系统变量比如class.path、os.name、user.dir等，也可以通过System.setProperty()方法设置自己需要的系统变量

3、refresh()

　 这个就是整个Spring Bean加载的核心了，它是ClassPathXmlApplicationContext的父类AbstractApplicationContext的一个方法，顾名思义，用于刷新整个Spring上下文信息，定义了整个Spring上下文加载的流程。



上面已经说了，refresh()方法是整个Spring Bean加载的核心，因此看一下整个refresh()方法的定义：

```
 1 public void refresh() throws BeansException, IllegalStateException {
 2         synchronized (this.startupShutdownMonitor) {
 3             // Prepare this context for refreshing.
 4             prepareRefresh();
 5 
 6             // Tell the subclass to refresh the internal bean factory.
 7             ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
 8 
 9             // Prepare the bean factory for use in this context.
10             prepareBeanFactory(beanFactory);
11 
12             try {
13                 // Allows post-processing of the bean factory in context subclasses.
14                 postProcessBeanFactory(beanFactory);
15 
16                 // Invoke factory processors registered as beans in the context.
17                 invokeBeanFactoryPostProcessors(beanFactory);
18 
19                 // Register bean processors that intercept bean creation.
20                 registerBeanPostProcessors(beanFactory);
21 
22                 // Initialize message source for this context.
23                 initMessageSource();
24 
25                 // Initialize event multicaster for this context.
26                 initApplicationEventMulticaster();
27 
28                 // Initialize other special beans in specific context subclasses.
29                 onRefresh();
30 
31                 // Check for listener beans and register them.
32                 registerListeners();
33 
34                 // Instantiate all remaining (non-lazy-init) singletons.
35                 finishBeanFactoryInitialization(beanFactory);
36 
37                 // Last step: publish corresponding event.
38                 finishRefresh();
39             }
40 
41             catch (BeansException ex) {
42                 // Destroy already created singletons to avoid dangling resources.
43                 destroyBeans();
44 
45                 // Reset 'active' flag.
46                 cancelRefresh(ex);
47 
48                 // Propagate exception to caller.
49                 throw ex;
50             }
51         }
52     }
```

每个子方法的功能之后一点一点再分析，首先refresh()方法有几点是值得我们学习的：

　　1、方法是加锁的，这么做的原因是避免多线程同时刷新Spring上下文

　　2、尽管加锁可以看到是针对整个方法体的，但是没有在方法前加synchronized关键字，而使用了对象锁startUpShutdownMonitor，这样做有两个好处：

　　（1）refresh()方法和close()方法都使用了startUpShutdownMonitor对象锁加锁，这就保证了在调用refresh()方法的时候无法调用close()方法，反之亦然，避免了冲突

　　（2）另外一个好处不在这个方法中体现，但是提一下，使用对象锁可以减小了同步的范围，只对不能并发的代码块进行加锁，提高了整体代码运行的效率

　　3、方法里面使用了每个子方法定义了整个refresh()方法的流程，使得整个方法流程清晰易懂。这点是非常值得学习的，一个方法里面几十行甚至上百行代码写在一起，在我看来会有三个显著的问题：

 　（1）扩展性降低。反过来讲，假使把流程定义为方法，子类可以继承父类，可以根据需要重写方法

 　（2）代码可读性差。很简单的道理，看代码的人是愿意看一段500行的代码，还是愿意看10段50行的代码？
　 （3）代码可维护性差。这点和上面的类似但又有不同，可维护性差的意思是，一段几百行的代码，功能点不明确，不易后人修改，可能会导致“牵一发而动全身”。
　　　　


　　　　
prepareRefresh方法：

下面挨个看refresh方法中的子方法，首先是prepareRefresh方法，看一下源码：


```
 1 /**
 2  * Prepare this context for refreshing, setting its startup date and
 3  * active flag.
 4  */
 5 protected void prepareRefresh() {
 6     this.startupDate = System.currentTimeMillis();
 7         synchronized (this.activeMonitor) {
 8         this.active = true;
 9     }
10 
11     if (logger.isInfoEnabled()) {
12         logger.info("Refreshing " + this);
13     }
14 }
```
这个方法功能比较简单，顾名思义，准备刷新Spring上下文，其功能注释上写了：

1、设置一下刷新Spring上下文的开始时间。

2、将active标识位设置为true。

另外可以注意一下12行这句日志，这句日志打印了真正加载Spring上下文的Java类。

obtainFreshBeanFactory方法：

obtainFreshBeanFactory方法的作用是获取刷新Spring上下文的Bean工厂，其代码实现为：


```
1 protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
2     refreshBeanFactory();
3     ConfigurableListableBeanFactory beanFactory = getBeanFactory();
4     if (logger.isDebugEnabled()) {
5         logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
6     }
7     return beanFactory;
8 }
```

其核心是第二行的refreshBeanFactory方法，这是一个抽象方法，有AbstractRefreshableApplicationContext和GenericApplicationContext这两个子类实现了这个方法，看一下上面ClassPathXmlApplicationContext的继承关系图即知，调用的应当是AbstractRefreshableApplicationContext中实现的refreshBeanFactory，其源码为：

 
```
 1 protected final void refreshBeanFactory() throws BeansException {
 2     if (hasBeanFactory()) {
 3         destroyBeans();
 4         closeBeanFactory();
 5     }
 6     try {
 7         DefaultListableBeanFactory beanFactory = createBeanFactory();
 8         beanFactory.setSerializationId(getId());
 9         customizeBeanFactory(beanFactory);
10         loadBeanDefinitions(beanFactory);
11         synchronized (this.beanFactoryMonitor) {
12             this.beanFactory = beanFactory;
13         }
14     }
15     catch (IOException ex) {
16         throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
17     }
18 }
```

这段代码的核心是第7行，这行点出了DefaultListableBeanFactory这个类，这个类是构造Bean的核心类，这个类的功能会在下一篇文章中详细解读，首先给出DefaultListableBeanFactory的继承关系图：

<a href="#">
    <img src="{{ site.baseurl }}/img/spring01-02.png" alt="Post Sample Image">
</a>



