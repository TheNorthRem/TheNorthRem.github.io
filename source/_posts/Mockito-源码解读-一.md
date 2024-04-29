---
title: Mockito 源码解读 (一)
date: 2024-04-29 16:09:23
tags:
---

# Mockito源码解读

## 前言

在bupt学计算机学到大三了，以前都是我拜读别人的博客，现在终于下定决心写点自己的的东西了，作为自己的第一篇正式上传到自己的博客网站的博客还是希望有一点含金量的。本人浑浑噩噩读到了大三下，转眼大三都过完了，技术没沉淀多少，学校学到的东西结完课就丢了。运气好靠着临时背的八股找到了一个实习，在实习的日子中能够在摸鱼的时候写点博客，增加点技术积累，希望能够将这个习惯在我的职业生涯中坚持久一点吧，别毕业后直接快进到码农烧烤就行了:)



最近的实习内容围绕着部门仓库进行单元测试建设，主要采用的Mock组件为Mockito，在深入实践和学习了Mockito后深感该组件对于编写单元测试的方便性并对其实现原理产生了好奇心因此在这对其源码进行学习

顺便吐槽一下对存量代码进行单元测试是真的折磨，大量的上下游接口调用，每个类需要Mock几十个方法，还要修正公司大模型生成的用例bug。希望遇见的业务代码都能有完善的单测

## 入口

这是我第一次对一个开源组件对源码进行解读，解读源码的第一步首先需要找到源码的入口，而根据Mockito的用法，在编写一个单测类时首先需要对单测类进行初始化。正常来讲，初始化所调用的接口为MocktioAnnatotations里的openMocks 函数

```java
@Before
public void setUp(){
    MockitoAnnotations.openMocks(this);
}
```

步入源码类内容如下

```java
public class MockitoAnnotations {
    public MockitoAnnotations() {
    }

    public static AutoCloseable openMocks(Object testClass) {
        if (testClass == null) {
            throw new MockitoException("testClass cannot be null. For info how to use @Mock annotations see examples in javadoc for MockitoAnnotations class");
        } else {
            AnnotationEngine annotationEngine = (new GlobalConfiguration()).tryGetPluginAnnotationEngine();
            return annotationEngine.process(testClass.getClass(), testClass);
        }
    }

    /** @deprecated */
    @Deprecated
    public static void initMocks(Object testClass) {
        try {
            openMocks(testClass).close();
        } catch (Exception var2) {
            Exception e = var2;
            throw new MockitoException(StringUtil.join(new Object[]{"Failed to release mocks", "", "This should not happen unless you are using a third-part mock maker"}), e);
        }
    }
}

```

在这段源码中，我们发现Annotations中有两个入口，分别是openMocks 和 initMocks 其中 initMocks 已被弃用，因此不再关注。



在openMocks 函数中首先进行了入参的非空判定，若判定非空后则调用了tryGetPluginAnnotationEngine接口尝试创建了一个引擎对象，并通过该引擎对象的process方法返回了一个AutoClosable的对象。这里先不考虑这些对象的作用，而是明确这个入口干了什么



## GlobalConfiguration

从入口代码可以看到，注解引擎是通过GlobalConfiguration 拿到的 那么GlobalConfiguration具体是什么东西呢 ，让我们步入它的源码

```java
public class GlobalConfiguration implements IMockitoConfiguration, Serializable {
    private static final long serialVersionUID = -2860353062105505938L;
    private static final ThreadLocal<IMockitoConfiguration> GLOBAL_CONFIGURATION = new ThreadLocal();

    IMockitoConfiguration getIt() {
        return (IMockitoConfiguration)GLOBAL_CONFIGURATION.get();
    }

    public GlobalConfiguration() {
        if (GLOBAL_CONFIGURATION.get() == null) {
            GLOBAL_CONFIGURATION.set(this.createConfig());
        }

    }

    private IMockitoConfiguration createConfig() {
        IMockitoConfiguration defaultConfiguration = new DefaultMockitoConfiguration();
        IMockitoConfiguration config = (new ClassPathLoader()).loadConfiguration();
        return (IMockitoConfiguration)(config != null ? config : defaultConfiguration);
    }

    public static void validate() {
        new GlobalConfiguration();
    }

    public AnnotationEngine getAnnotationEngine() {
        return ((IMockitoConfiguration)GLOBAL_CONFIGURATION.get()).getAnnotationEngine();
    }
		....
}

```

可以看到，在GlobalConfiguration中还包了一层IMockitoConfiguration对象，它是使用ThreadLocal进行保存的，这个ThreadLocal采用静态修饰，因此只会初始化一次，对于ThreadLocal的作用在于其get的到的值能通过哈希的方式与线程相对应，因此每个线程从其拿到的对象是跟该线程一一对应的。



根据其构造函数可以看到，每当建立一个GlobalConfiguration对象时，都会通过当前线程判断该线程是否已经在ThreadLocal中创建了一个IMockitoConfiguration，对于每个线程只会创建一个IMockitoConfiguration对象，这是一种单例设计模式(懒汉式)，因为每个线程可能创建多个GlobalConfiguration对象，通过这种方式可以保证线程内IMockitoConfiguration对象的唯一性。



对于如何创建IMockitoConfiguration对象，这里不关心与核心原理无关。

可以看到，在getAnnotationEngine() 方法中调用了GLOBAL_CONFIGURATION中的IMockitoConfiguration的getAnnotationEngine方法，而该方法的实现类位于同样实现了IMockitoConfiguration 接口的 DefaultMockitoConfiguration 类中



```java
public class DefaultMockitoConfiguration implements IMockitoConfiguration {
    public DefaultMockitoConfiguration() {
    }

    public Answer<Object> getDefaultAnswer() {
        return new ReturnsEmptyValues();
    }

    public AnnotationEngine getAnnotationEngine() {
        return new InjectingAnnotationEngine();
    }

    public boolean cleansStackTrace() {
        return true;
    }

    public boolean enableClassCache() {
        return true;
    }
}

```



在该类的getAnnotationEngine 方法中创建了一个InjectingAnnotationEngine 对象，可以认为该对象就是入口处

```java
(new GlobalConfiguration()).tryGetPluginAnnotationEngine()
```

调用的返回值

## InjectingAnnotationEngine

回顾一下在入口处干了什么事情，入口处调用了annotationEngine 的process函数传入的参数分别为调用openMock类的类型和类实例，这里可以认为就是单测类本身。

```java
public AutoCloseable process(Class<?> clazz, Object testInstance) {
        List<AutoCloseable> closeables = new ArrayList();
        closeables.addAll(this.processIndependentAnnotations(testInstance.getClass(), testInstance));
        closeables.addAll(this.processInjectMocks(testInstance.getClass(), testInstance));
        return () -> {
            Iterator var1 = closeables.iterator();

            while(var1.hasNext()) {
                AutoCloseable closeable = (AutoCloseable)var1.next();
                closeable.close();
            }

        };
}
```



可以看到，在该函数中主要干了三件事：

1. 处理单测类中的普通注解(如 @Mocks,@Spy )
2. 处理单测类中的InjectMocks注解
3. 将处理注解后返回的AutoCloseable 对象打包成一个AutoCloseable对象进行返回

对于第三件事没什么好解读的，接下来我们按顺序对前两件事的完成逻辑进行解读



### processIndependentAnnotations

```java
private List<AutoCloseable> processIndependentAnnotations(Class<?> clazz, Object testInstance) {
  List<AutoCloseable> closeables = new ArrayList();

  for(Class<?> classContext = clazz; classContext != Object.class; classContext = classContext.getSuperclass()) {
    closeables.add(this.delegate.process(classContext, testInstance));
    closeables.add(this.spyAnnotationEngine.process(classContext, testInstance));
  }

  return closeables;
}
```

这个函数干的事情也很简明(突然让我感叹大佬的代码是真的优雅，让我读起来如此丝滑)



通过for循环中的条件可以判断，其从传入的clazz类开始向上遍历它的父类，同时分别从delegate和spyAnnotationEngine对象调用process方法对遍历的类进行处理，根据命名可以推测出处理的分别为@Mocks 和 @Spy 注解。



让我们回到InjectingAnnotationEngine的声明代码中

```java
public class InjectingAnnotationEngine implements AnnotationEngine, org.mockito.configuration.AnnotationEngine {
    private final AnnotationEngine delegate = new IndependentAnnotationEngine();
    private final AnnotationEngine spyAnnotationEngine = new SpyAnnotationEngine();

    public InjectingAnnotationEngine() {
    }
    
    ...
}
```



可以看到，InjectingAnnotationEngine 为 AnnotationEngine 的一个实现类，同时它拥有两个私有对象 delegate 和 spyAnnotation 分别为AnnotationEngine的另外两个实现类，分别为IndependentAnnotationEngine、和SpyAnnotationEngine，而上文中调用的process方法是在这两个实现类中实现的。



## IndependentAnnotationEngine



由上文知，在这个实现类中实现了对@Mock 注解的处理逻辑，主要在其实现的process函数中，其源码如下：



```java
public AutoCloseable process(Class<?> clazz, Object testInstance) {
        List<MockedStatic<?>> mockedStatics = new ArrayList();
        Field[] fields = clazz.getDeclaredFields(); //获取类中声明的字段
        Field[] var5 = fields;
        int var6 = fields.length;

        for(int var7 = 0; var7 < var6; ++var7) { //对字段进行遍历
            Field field = var5[var7];
            boolean alreadyAssigned = false;
            Annotation[] var10 = field.getAnnotations();
            int var11 = var10.length;

            for(int var12 = 0; var12 < var11; ++var12) {
                Annotation annotation = var10[var12];
                Object mock = this.createMockFor(annotation, field);
                if (mock instanceof MockedStatic) {
                    mockedStatics.add((MockedStatic)mock);
                }

                if (mock != null) {
                    this.throwIfAlreadyAssigned(field, alreadyAssigned); //若alreadyAssigned则说明重复注解 抛出异常
                    alreadyAssigned = true;

                    try {
                        FieldSetter.setField(testInstance, field, mock);
                    } catch (Exception var18) {
                        Exception e = var18;
                        Iterator var16 = mockedStatics.iterator();

                        while(var16.hasNext()) {
                            MockedStatic<?> mockedStatic = (MockedStatic)var16.next();
                            mockedStatic.close();
                        }

                        throw new MockitoException("Problems setting field " + field.getName() + " annotated with " + annotation, e);
                    }
                }
            }
        }

        return () -> {
            Iterator var1 = mockedStatics.iterator();

            while(var1.hasNext()) {
                MockedStatic<?> mockedStatic = (MockedStatic)var1.next();
                mockedStatic.closeOnDemand();
            }

        };
    }
```



这段代码相对之前而言比较复杂，但我们根据注解的作用机理可以知道，它要对类内的注解进行处理，首先需要通过反射的方式拿到类中的Fields字段集合，再遍历Fields集合拿到类中声明的属性，再通过属性拿到属性上的注解才能对其进行操作。



类的开头声明了MockedStatic的集合，在方法的结尾处我们可以看到，方法的返回值是对该集合中的MockedStatic对象进行close封装，因此可以看出，openMock 中需要回收的对象主要为MockedStatic对象，而为什么要对MockStatic对象进行回收处理呢，这与MockStatic的生命周期有关了，后文应该会对其进行探讨，如果我没忘记的话..



话说回来，我们可以看到，在下一步中，代码通过clazz对象调用getDeclaredFields方法拿到了类中声明的字段集合以及集合的长度并对其进行了遍历。

在遍历的过程中，我们根据拿到的Field可以访问到其标识的注解。在第二层循环中，我们对注解进行遍历，并调用createMockFor方法尝试对注解的字段创建Mock



createMockFor方法中执行的逻辑如下：

```java
private Object createMockFor(Annotation annotation, Field field) {
  return this.forAnnotation(annotation).process(annotation, field);
}

private <A extends Annotation> FieldAnnotationProcessor<A> forAnnotation(A annotation) {
  return this.annotationProcessorMap.containsKey(annotation.annotationType()) ? (FieldAnnotationProcessor)this.annotationProcessorMap.get(annotation.annotationType()) 
    : 
  new FieldAnnotationProcessor<A>() {
    public Object process(A annotation, Field field) {
      return null;
    }
  };
}
```

可以看到，在createMockFor方法中调用了forAnnotation方法，forAnnotation方法从annotationProcessorMap中查询是否包含传入的注解类型，若存在的话根据该注解类型拿到对应的Processor(处理器) 否则的话就创建一个process函数返回值为空的处理器



因此调用createMockFor 函数一般有两种情况，一种是注解类型为annotationProcessorMap 中存在的类型，这种情况会进入对应处理器的处理逻辑并返回对应的对象，另一种情况为注解不存在annotationProcessorMap中存在的类型（即和Mock无关的注解），则调用返回值为null



同时在下文代码中不难看到，若返回值mock为null的话不会进入后续的处理分支。



让我们先把循环中的逻辑看完，通过createMockFor 函数拿到了对应的返回值后，我们先对其进行了类型判断，若它为MockStatic类型则将其加入到mockedStatics集合中，方便对其进行资源回收；在下文中，我们对其进行非空校验，若非空则说明该注解是与Mock相关的注解，对于一个字段只能Mock一次和一种注解，因此通过alreadyAssigned标识能够判定其是否只注解过一次，因此在遍历注解的时候会通过throwIfAlreadyAssigned进行判定，若重复注解Mock相关的内容则会抛出异常。

```java
void throwIfAlreadyAssigned(Field field, boolean alreadyAssigned) {
    if (alreadyAssigned) {
        throw Reporter.moreThanOneAnnotationNotAllowed(field.getName());
    }
}
```



在接下来的处理中，函数调用了FieldSetter.setField函数进行字段处理，同时进行了一个异常的捕获，使其在处理过程中如果发生异常后能够及时关闭在上文循环中创建的MockStaticed的对象并抛出另一个异常来提醒编程人员这里出错了。



这个函数的作用解读完了，接下来我们可以分别从FieldSetter.setField函数和annotationProcessorMap中的处理器进行入手，写不完了写在下一章
