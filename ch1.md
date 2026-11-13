# 第1章 全局地图：一次 getBean 穿过哪几层

**一句话**：容器拿到的是一份 `BeanDefinition` 说明书，交给你的是可能已被换成代理的成品对象；中间隔着「造壳 → 填属性 → 初始化 → 换壳 → 入池」五道关。

**为什么需要它**：Spring 里的对象不是你 `new` 出来的，所以你无法用「看构造器和 setter」的办法推断它此刻的状态。同一个类，在填充属性之前是半成品，在初始化之后可能被整个替换成代理；你在 `@PostConstruct` 里拿到的 `this`，和别人 `@Autowired` 拿到的那个对象，未必是同一个。不建立这条时序，所有「注解写了为什么不生效」都只能靠猜。

**核心结构**：

| 层 | 关键组件（语料中的名字） | 职责 | 出错时的样子 |
|---|---|---|---|
| 元信息 | `BeanDefinition`、`BeanDefinitionRegistry` | 记下造什么、怎么造 | 改配置不生效、候选者冲突 |
| 生产 | `DefaultListableBeanFactory`、`doGetBean` | 按说明书造 Bean | `BeanCurrentlyInCreationException` |
| 缓存 | `DefaultSingletonBeanRegistry` 的三个 Map | 存放成品 / 半成品 / 工厂 | 循环依赖解不开 |
| 填充 | `populateBean`、`@Autowired` | 把依赖注入半成品 | 字段为 null、多候选报错 |
| 初始化 | `initializeBean`、`BeanPostProcessor` | 回调、初始化、包装 | 代理没生成、`@PostConstruct` 时机错 |
| 增强 | `AnnotationAwareAspectJAutoProxyCreator` | 生成 AOP / 事务代理 | 切面、事务都不生效 |
| 收尾 | `DisposableBean`、`@PreDestroy` | 容器关闭时释放 | 连接池、线程池没关 |

**关键设计**：真正决定 Spring 行为的不是「有哪些注解」，而是**两条不可调换的时序**——属性填充在实例化之后、初始化之前；代理替换在初始化之后。

```text
【一次 getBean 的六步】
  doGetBean
    │ ① getSingleton：先查一级缓存 singletonObjects
    ▼
  ② 没命中 → createBean → createBeanInstance（只有壳）
    │ ③ addSingletonFactory：ObjectFactory 进三级缓存
    ▼
  ④ populateBean：按名称/类型找依赖，填进壳里
    │ ⑤ initializeBean：Aware → before → init → after
    ▼
  ⑥ 成品（或代理）入一级缓存，返回给调用方

【两条时序约束】
  实例化 ── 早于 ── 属性填充       → 构造器注入没法用半成品
  属性填充 ── 早于 ── 初始化       → 初始化方法里字段才齐
  初始化 ── 早于 ── 代理生成       → @PostConstruct 里 this 不是代理
```

**拆解**：

### 一次 getBean 的调用栈长什么样

语料把单例 Bean 的创建入口定位在 `AbstractBeanFactory#doGetBean`：`doGetBean` 先调 `getSingleton` 查缓存，能取到就直接返回，取不到才进入创建流程。创建流程里依次是 `createBeanInstance` 得到实例包装、`addSingletonFactory` 把 `ObjectFactory` 放进三级缓存、`populateBean` 填充属性、`initializeBean` 完成初始化与后置处理，最后成品进 `singletonObjects`。这一段在语料的「循环依赖」专题里被逐行注释过，是全书最值得记住的一条主干。

### 为什么要有「半成品」这个概念

`createBeanInstance` 之后的对象，字段都还是默认值。它能被提前暴露出去，是因为 Spring 允许「先给引用、后补内容」：把能产出早期引用的 `ObjectFactory` 存进三级缓存，别的 Bean 需要它时先拿这个引用来完成自己的属性填充。这也是循环依赖能被解开的前提。**半成品不是异常状态，而是设计中的合法中间态**，只是它意味着你若在这个阶段用它做业务判断，读到的是没填好的字段。

### 容器里其实有三种「对象」

第一种是原始实例，刚 `createBeanInstance` 出来；第二种是早期引用，可能被 `getEarlyBeanReference` 加工过（语料注释明确写了这一步「可以在这个里面通过 aop 生成代理对象」）；第三种是最终暴露对象，来自 `initializeBean` 的返回值 `exposedObject`。语料里那段 `getSingleton(beanName, false)` 与 `exposedObject` 的比对，本质上就是在保证「别人提前拿走的引用」和「最后放进池子的对象」是同一个，否则循环依赖解开了却埋下对象不一致的坑。

### 从配置到销毁的完整账本

语料按 13 个阶段展开 Bean 生命周期（文中称「12 个环节」，因为实例化、属性赋值、初始化几个阶段内部又细分了小阶段）：元信息配置 → 元信息解析 → 注册到容器 → `BeanDefinition` 合并 → Bean Class 加载 → 实例化（实例化前 / 实例化）→ 合并后的 BeanDefinition 处理 → 属性赋值（实例化后 / 赋值前 / 赋值）→ 初始化（Aware 回调 / 初始化前 / 初始化 / 初始化后）→ 所有单例初始化完成 → 使用 → 销毁前 → 销毁。前 10 段归容器启动，第 11 段归业务，最后两段归容器关闭。第 3 到第 9 章就是把这份账本逐段拆开。

**今天怎么做**（延伸，非语料原文）：现代 Spring Boot 把 `refresh()` 藏进自己的启动入口里，启动阶段还多了配置绑定、条件评估与自动配置导入；Spring 6 / Boot 3 之后又引入 AOT 与原生镜像的路径，把一部分「运行时扫描 + 反射」提前到构建期。容器对 Bean 生命周期的编排没变，但**说明书是在什么时候、由谁生成**这一层已经和语料时代不同。把这些当作后来的演进，别拿它去解释 5.2.x 的源码行为。

<div class="chapend" markdown="1">

**容易误解的地方**：
1. 「`getBean` 就是 `new` 一个对象」——它先查缓存，命中直接返回；没命中才走创建，且返回值可能是代理而不是原始实例。
2. 「`@Autowired` 之后对象就完全可用了」——属性填充只是第 ④ 步，初始化方法、`BeanPostProcessor` 还没跑，此刻的 `this` 还不是最终形态。
3. 「 `@PostConstruct` 里调用自己的另一个 `@Transactional` 方法会有事务」——代理在初始化后阶段才生成，且 `this` 本来就不经过代理。
4. 「循环依赖是 Spring 的 bug」——它是半成品提前暴露带来的能力，构造器注入之所以无解，恰恰因为造壳这一步没法提前给引用。
5. 「容器只管创建不管销毁」——对容器管理的单例，关闭时会调用 `DisposableBean`、`@PreDestroy` 和指定的销毁方法，并清理缓存；但 prototype 不在这套账里。

**记住这一句**：`getBean` 是一条「查缓存 → 造壳 → 填属性 → 初始化 → 换代理 → 入池」的流水线，属性填充夹在实例化与初始化之间、代理生成排在初始化之后，Spring 里大半的「不生效」都源于踩错这两条时序。

</div>
