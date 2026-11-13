# 第2章 容器骨架：BeanFactory、ApplicationContext 与 refresh

**一句话**：`BeanFactory` 是生产 Bean 的工厂，`ApplicationContext` 是在工厂外面加了一层启动编排的上下文；`refresh()` 把「读配置 → 改说明书 → 注册后置处理器 → 造单例 → 发事件」固化成固定顺序的 13 个阶段。

**为什么需要它**：只知道 `getBean` 的六步，还解释不了「为什么某些扩展点能改配置、某些只能改实例」「为什么监听器在有些时机收不到事件」。原因是这些能力被挂在了容器启动的不同阶段上，而阶段的顺序是写死的。把 `refresh` 的阶段表读懂，你就能判断一个扩展点最早能在什么时候生效。

**核心结构**：

| 维度 | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| 定位 | Bean 的注册、创建、查找 | 在 BeanFactory 之上加生命周期编排 |
| 语料里的代表 | `DefaultListableBeanFactory` | `AnnotationConfigApplicationContext` |
| 典型能力 | `getBean`、`registerSingleton`、BeanDefinition 注册 | `refresh`、事件广播、国际化 `MessageSource` |
| 创建时机 | `GenericApplicationContext` 构造器里就 `new` 好 | 构造器创建上下文与读取器，`refresh()` 才启动 |
| 谁造单例 | 惰性，取用到才造 | `refresh` 阶段 12 主动实例化非懒加载单例 |

**关键设计**：`refresh()` 的阶段顺序本身是一条约束——**改说明书的处理器必须先跑，改实例的后置处理器必须先注册，最后才造单例**。任何一步提前，后面的人就看不到前一步的产物。

```text
【refresh 的 13 个阶段（语料源码注释口径）】
 1 创建 Spring 应用上下文
 2 应用上下文启动准备           prepareRefresh
 3 BeanFactory 创建             obtainFreshBeanFactory
 4 BeanFactory 准备             prepareBeanFactory
 5 BeanFactory 后置处理         postProcessBeanFactory
 6 执行 BeanFactoryPostProcessor invokeBeanFactoryPostProcessors
 7 注册 BeanPostProcessor       registerBeanPostProcessors
 8 初始化内建 Bean：MessageSource
 9 初始化内建 Bean：事件广播器   initApplicationEventMulticaster
10 上下文刷新（子类实现）       onRefresh
11 注册事件监听器               registerListeners
12 实例化所有非 lazy 单例       finishBeanFactoryInitialization
13 刷新完成                     finishRefresh
```

**拆解**：

### 上下文是怎么被造出来的

语料用 `AnnotationConfigApplicationContext` 做例子：调用子类构造器时先走父类 `GenericApplicationContext` 的构造器，里面直接 `this.beanFactory = new DefaultListableBeanFactory()`；`AnnotationConfigApplicationContext` 自己的构造器再创建 `AnnotatedBeanDefinitionReader`，并向容器注册几个关键的内建 Bean。也就是说**工厂先于配置存在**，配置是往已经存在的工厂里登记，而不是配置去创建工厂。这也解释了为什么「注册」和「实例化」能分成两个阶段。

### 前六个阶段在忙什么

阶段 2 做启动准备并标记状态，阶段 3 拿到 BeanFactory，阶段 4 给工厂装好基础设置（类加载器、表达式解析器等基础设施），阶段 5 是留给子类的 `postProcessBeanFactory`，阶段 6 调用 `invokeBeanFactoryPostProcessors`——这是 `BeanFactoryPostProcessor` / `BeanDefinitionRegistryPostProcessor` 唯一能改动 `BeanDefinition` 的窗口。语料把这几步概括为「Bean 注册阶段 → BeanFactory 后置处理阶段」，关键结论是：**说明书在这一段还是可写的**。

### 阶段 7 到 13 为什么这样排

阶段 7 `registerBeanPostProcessors` 把会干预实例创建的处理器注册好，但此时还没有批量造业务 Bean；阶段 8、9 先把 `MessageSource` 和 `applicationEventMulticaster` 这两个内建 Bean 准备好；阶段 10 留给子类；阶段 11 `registerListeners` 登记监听器；阶段 12 `finishBeanFactoryInitialization` 才实例化所有剩余的、非 `lazy init` 的单例；阶段 13 `finishRefresh` 收尾。语料在事件专题里专门解释了「早期事件」：广播器还没建好时发出的事件会暂存在 `earlyApplicationEvents`，等广播器就绪再补发——这正是阶段 9 必须早于阶段 11、12 的原因。

### 顺序错了会怎样

假设把阶段 12 提到阶段 6 之前：单例已经造完，`BeanFactoryPostProcessor` 再改 `BeanDefinition` 就只对「下一次创建」有效，对已经进池的对象毫无作用。假设把阶段 7 挪到阶段 12 之后：前面造出来的 Bean 全都绕过了 `BeanPostProcessor`，AOP 与事务代理不会生成，而这些恰恰是本书第 6、9、10 章的全部内容。所以 `refresh` 不是一串可随意增删的钩子，而是**把「能改配置 → 能改实例 → 开始造实例」三个时代切开的隔板**。

**今天怎么做**（延伸，非语料原文）：Spring Boot 的启动把这套 `refresh` 包进了更长的流程里，条件注解与自动配置导入集中在阶段 6 前后完成，因此「配置类有没有被解析」成了排查自动配置问题的第一问；Spring 6 / Boot 3 的 AOT 路线又把一部分扫描与注册挪到构建期，运行时的 `refresh` 更接近「按预生成说明书执行」。这些是后来的演进，不改变 5.2.x 源码里 13 个阶段的先后关系。

<div class="chapend" markdown="1">

**容易误解的地方**：
1. 「`ApplicationContext` 是另一种 BeanFactory 实现」——它内部持有一个 `DefaultListableBeanFactory`，是在其上叠加生命周期与企业级能力的上下文。
2. 「`refresh()` 可以调多次」——语料把它当作一次启动流程展开，重复刷新会牵涉销毁与重建；日常应把它理解为「一次启动、一次编排」。
3. 「`BeanFactoryPostProcessor` 和 `BeanPostProcessor` 差不多」——前者在阶段 6 改 `BeanDefinition`，后者在阶段 7 注册、作用于每个实例；改的对象和时机都不同。
4. 「容器一启动就把所有 Bean 都造出来」——阶段 12 只造非 `lazy init` 的单例，标了 `@Lazy` 的与 prototype 都要等取用时才创建。
5. 「发事件一定立刻有人收到」——广播器未就绪时的事件属于早期事件，会暂存到广播器准备好之后才广播。

**记住这一句**：`refresh` 的 13 个阶段把启动切成「可改说明书（阶段 6）→ 可改实例（阶段 7）→ 开始造单例（阶段 12）」三段，扩展点能不能生效，只看它被挂在哪一段。

</div>
