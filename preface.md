## 序：为什么读这本 Spring

写 Spring 写得越久，越会发现「注解没写错、SQL 也没错、数据却没回滚」的根，常常不在你的业务代码里，而在你手里那个对象根本不是你 `new` 出来的那个。容器替你造了它、填了它的字段、又可能在外面套了一层代理——你调用的每一次方法，都要先穿过这层壳。看不懂这层壳，`@Transactional` 为什么时灵时不灵、循环依赖为什么有时报有时不报、Bean 为什么在 `@PostConstruct` 里还是半成品，就永远只能靠试。

> **定位与版本口径**：主口径是 **Spring Framework 5.2.x 时代的注解驱动容器**——三份语料分别是《Spring 笔记》（697 页）与《Spring 系列（路人甲 Java）》上、下册（471 + 472 页），合计约 **205 万字**；案例环境在语料里明确标注为 `jdk1.8 / Spring 5.2.3.RELEASE / mysql5.7`。骨架取自「Bean 生命周期 12 个环节」「应用上下文 refresh 阶段」「循环依赖三级缓存」「AOP 核心源码」「事务 7 种传播行为」「事务失效 7 种情况」这些成体系的专题。Spring 6 / Spring Boot 3 的变化只出现在每章末尾的「今天怎么做」，已明确标注为**延伸**，不与语料原文混同。
>
> **这本书不写什么**：三份语料里 `doDispatch`、`HandlerMapping`、`HandlerAdapter` 的命中为零，因此**不写 Spring MVC 的源码深挖**——那不是这批资料能支撑的内容，硬写就是编造。与已归档书的分工：《Spring Cloud 微服务实战》讲一次请求在微服务链路里怎么走，《SpringBoot核心原理》讲自动配置与启动，《Spring揭秘_精华版》是框架整体的入门地图；本书只盯着**一个 Bean 从配置到销毁**，以及为它服务的容器、依赖、代理、事务与扩展点。

主线只有一条：**一份配置变成 BeanDefinition，BeanDefinition 变成半成品，半成品被填属性、被初始化、可能被换成代理，最后进单例池；容器关闭时再走一遍销毁回调**。把这条线读薄，你就能把「玄学不生效」翻译成一句可验证的判断：现在拿到的，到底是原始对象、半成品，还是代理对象。

## 一、一个 Bean 的一生链路速查

```text
配置来源：@Component / @Bean / XML / API
  │  ① 解析、注册 → BeanDefinition（一份「生产说明书」）
  ▼
refresh() → finishBeanFactoryInitialization
  │  ② 实例化非懒加载单例：doGetBean → getSingleton → createBean
  ▼
  ③ createBeanInstance：得到只有壳的「半成品」对象
  │  ④ addSingletonFactory：把 ObjectFactory 放进三级缓存
  ▼
  ⑤ populateBean：按 @Autowired / @Resource 找到依赖并填进去
  │     若依赖也在创建中 → 从三级缓存取早期引用（可解循环依赖）
  ▼
  ⑥ initializeBean
       Aware 回调 → BPP before → 初始化方法 → BPP after
       最后一步常被换成 AOP / 事务代理对象
  ▼
  ⑦ 成品进 singletonObjects（一级缓存），业务拿到的是它
  │
  ▼
  ⑧ 容器 close → DisposableBean / @PreDestroy / 销毁方法
```

关键只有一句：**属性填充发生在实例化之后、初始化之前；代理发生在初始化之后**。事务、AOP、循环依赖这三类经典问题，全都是这两条时序约束的推论。

## 二、12 章一句话速查

| 章 | 一句话 |
|---|---|
| 1 全局地图 | 一次 `getBean` 走的是「查缓存→造壳→填属性→初始化→换代理→入池」 |
| 2 容器骨架 | `BeanFactory` 管生产，`ApplicationContext` 用 `refresh` 把生产编排成 13 个阶段 |
| 3 BeanDefinition | 所有配置最终都变成一份说明书，容器照着它造，不照着你的类造 |
| 4 实例化 | 造壳这一步有前后扩展点，`FactoryBean` 甚至能换掉「造出来的东西」 |
| 5 属性填充 | 依赖注入发生在半成品阶段，`@Primary` / `@Qualifier` 决定候选者怎么选 |
| 6 初始化 | Aware → 初始化前 → 初始化 → 初始化后，AOP 代理在最后一步换壳 |
| 7 三级缓存 | 三个 Map 分工商成品、半成品、工厂，第三级是为了让代理能提前生成 |
| 8 作用域与销毁 | singleton 才进缓存池，prototype 容器只负责造、不负责收 |
| 9 AOP | 连接点 / 切入点 / 通知 / 顾问四件套，最终落成 JDK 或 CGLIB 代理 |
| 10 声明式事务 | 事务是套在方法上的一层拦截器，7 种传播行为决定嵌套时谁跟谁 |
| 11 事件与扩展点 | 容器在关键节点留了钩子，`BeanFactoryPostProcessor` 改说明书、BPP 改实例 |
| 12 排查与边界 | 先问「拿到的是代理吗」「在同一个线程吗」，再翻异常与配置 |

## 三、症状 → 章节

| 你看到的现象 | 先翻哪章 | 根因类型 |
|---|---|---|
| `@Transactional` 完全不回滚 | 6、9、10、12 | 没走代理 / 自身调用 / 异常类型 |
| 同类里 `this.m2()` 事务失效 | 9、10、12 | 调用绕过了代理对象 |
| 启动报循环依赖 | 5、7 | 构造器注入 / prototype 参与 |
| `BeanCurrentlyInCreationException` | 7 | 三级缓存没能提供早期引用 |
| `@Autowired` 找到多个实现报错 | 3、5 | 候选者冲突，`@Primary` / `@Qualifier` |
| 注入进来的对象字段全是 null | 4、5、6 | 在实例化/填充之前就提前使用了 Bean |
| `@PostConstruct` 里拿不到代理 | 6、9 | 代理在初始化后阶段才生成 |
| `FactoryBean` 取到的类型不对 | 4 | 取的是 `getObject()` 产物，不是工厂本身 |
| 切面不生效、通知顺序乱 | 9 | 切入点没匹配 / Advisor 顺序 / 代理方式 |
| 单例里注入 prototype 却只创建一次 | 8 | 作用域语义与注入时机 |
| 想改配置但改不动已创建的 Bean | 3、11 | 改 BeanDefinition 要在实例化之前 |
| 容器关闭时资源没释放 | 8 | 销毁回调只对容器管理的单例生效 |
| 事件发出去没人收到 | 11 | 广播器/监听器注册时序、早期事件 |
| 事务里 RPC 变慢、连接被占住 | 10、12 | 大事务，事务与业务不在同一边界 |

## 四、20 个自测问题

1. 从 `@Component` 到你能 `getBean` 拿到对象，中间必须经过哪几步？
2. `BeanFactory` 和 `ApplicationContext` 的分工差在哪？为什么后者能做事件与国际化？
3. `refresh()` 里「注册 BeanPostProcessor」和「实例化所有非懒加载单例」谁先谁后？为什么？
4. 一份 `BeanDefinition` 至少记了哪些信息？`scope`、`lazyInit`、`dependsOn`、`primary` 各管什么？
5. 「BeanDefinition 合并」是什么意思？`parent` 配置为什么需要它？
6. 实例化前扩展点能做什么？它能让容器彻底不 `new` 这个类吗？
7. `FactoryBean` 和普通 Bean 有什么区别？想拿工厂本身该怎么写 bean 名称？
8. `populateBean` 发生在初始化之前还是之后？这个顺序决定了什么？
9. `@Autowired`、`@Resource`、`@Primary`、`@Qualifier` 分别在候选者选择里扮演什么角色？
10. Bean 初始化的四个小阶段是什么？`BeanPostProcessor` 的两个方法各挂在哪一段？
11. AOP 代理为什么常在 `postProcessAfterInitialization` 里生成，而不是更早？
12. 三级缓存的三个 Map 分别叫什么、各存什么？为什么要第三级而不是两级？
13. 构造器循环依赖为什么无解，setter 循环依赖为什么能解？
14. `singletonsCurrentlyInCreation` 是干什么的？没有它会怎样？
15. singleton 与 prototype 在「容器是否负责销毁」上有什么本质区别？
16. Spring AOP 里 JoinPoint / Pointcut / Advice / Advisor / Aspect 五个词怎么区分？
17. 什么条件下选 JDK 动态代理、什么条件下选 CGLIB？`proxyTargetClass` 起什么作用？
18. `@Transactional` 的 7 种传播行为里，哪几种会挂起当前事务？
19. 事务失效的 7 种情况里，哪一条和「代理」直接相关？
20. 默认回滚的异常类型是什么？异常被 catch 吞掉为什么事务还是提交了？

## 五、一个提醒

这本书只讲「机制与判断」，不替你背注解参数。同一份机制，在 Spring 5.2 与 Spring 6 里的默认行为、在 XML 与注解驱动下的写法都不一样；**把章节里的时序当成心智模型，把注解属性当成随版本而变的旋钮**。遇到「不生效」，先按第 12 章的顺序问三句话：这个对象是不是容器造的？我调用它时有没有经过代理？当前代码和业务是不是在同一个线程里？多数玄学，都能在这三句话里现形。