# 第3章 BeanDefinition：配置怎么变成一份生产说明书

**一句话**：不管是 XML 的 `<bean>`、`@Bean`、`@Component` 还是直接调 API，最终都会被解析成同一份 `BeanDefinition`；容器造对象时看的是这份说明书，不是你的源码。

**为什么需要它**：既然容器不 `new` 对象，它就必须知道「用哪个类、是不是单例、要不要延迟、先造谁、能不能被自动注入、初始化调哪个方法」。这些信息如果散落在注解里，容器就无法在创建前统一修改、合并、校验。把配置收敛成一份数据结构，才有了 `@Conditional` 的条件注册、`BeanFactoryPostProcessor` 的统一改写，以及父子配置继承。

**核心结构**：

| `BeanDefinition` 里的信息 | 语料里的对应接口方法 / 字段 | 影响哪一步 |
|---|---|---|
| 类名 | `getBeanClassName()` | 阶段 5 加载 Class、阶段 6 实例化 |
| 作用域 | `setScope` / `getScope`（singleton、prototype 等） | 是否进单例缓存、是否管销毁 |
| 延迟初始化 | `setLazyInit` / `isLazyInit`（只对单例有效） | 阶段 12 是否跳过 |
| 依赖顺序 | `getDependsOn` / `setDependsOn` | 保证被依赖的 Bean 先初始化 |
| 注入候选资格 | `setAutowireCandidate` / `isAutowireCandidate` | `@Autowired` 选候选时是否被排除 |
| 主要候选者 | `primary` | 多个候选时优先选谁 |
| 父定义 | `setParentName` / `getParentName` | 阶段 4 的 `BeanDefinition` 合并 |
| 初始化 / 销毁方法 | 定义 Bean 时指定的初始化方法、销毁方法 | 阶段 9 与阶段 13 |

**关键设计**：**四种配置入口，一个出口**。语料明确写了「Bean 信息定义 4 种方式：API、XML、properties、注解」，并强调「其他几种方式最终都会采用 API 的方式来定义 bean 配置信息」；无论 `<bean>` 标签、`@Bean`、`@Component` 还是扫描出来的类，最后都变成 `BeanDefinition` 对象交给 Bean 工厂。

```text
【配置入口 → 唯一出口】
 XML <bean>  ─┐
 @Bean       ─┤
 @Component ──┼→ 解析 → BeanDefinition → BeanDefinitionRegistry
 @ComponentScan  │        （说明书）         （登记册）
 properties / API ┘
                       │
                       ▼  阶段4：合并（parent 继承）
                       ▼  阶段6：BeanFactoryPostProcessor 可改
                       ▼  阶段12：工厂照它造实例
```

**拆解**：

### 四种入口分别解决什么

XML 与 properties 属于外部化配置，改配置不用重新编译；`@Component` 加 `@ComponentScan` 解决「批量登记」，语料里有专门讲 `@ComponentScan`、`@Import`、`@Conditional` 的批量注册专题；`@Bean` 适合把构造过程写在方法里，还能顺手指定初始化方法；API 方式最直接，语料里能看到 `DefaultListableBeanFactory` 直接 `registerSingleton` 的用法。它们的差别只在「怎么把信息写进来」，**造对象的逻辑完全共用**。

### 说明书里为什么要有这么多字段

`scope` 决定要不要进单例池，`lazyInit` 决定阶段 12 是否跳过，`dependsOn` 决定创建顺序，`autowireCandidate` 与 `primary` 决定它在别人的注入候选里排第几，`parent` 决定合并时从谁继承。这些字段的共同点是：**都必须在对象被创建之前就确定**。一旦实例已经生成，再改 `scope` 或 `lazyInit` 就没有意义了。这正是「配置阶段」与「实例化阶段」必须分开的原因。

### 注册与合并这两步在做什么

注册是把说明书放进 `BeanDefinitionRegistry`，此时还没有任何业务对象被造出来。合并（阶段 4）处理的是父子定义：子定义可以只写差异项，其余从 `parent` 继承，容器实际使用的是「合并后的 BeanDefinition」。语料的生命周期清单里把「合并后的 BeanDefinition 处理」单独列为阶段 7，并对应到 `MergedBeanDefinitionPostProcessor`——说明合并后的那份说明书，才是后续实例化与属性填充真正读取的版本。

### 谁能改说明书、改到什么时候为止

`BeanFactoryPostProcessor` 与 `BeanDefinitionRegistryPostProcessor` 是改说明书的正规入口，语料里专门有「BeanFactory 扩展」专题讲这两个接口；后者还能往注册表里新增定义，因此常和 `@Import`、`ImportSelector`、`@Conditional` 的批量注册一起出现。它们的执行窗口是 `refresh` 阶段 6，早于阶段 7 注册 `BeanPostProcessor`、更早于阶段 12 造单例。过了阶段 12 再去改 `BeanDefinition`，对已入池的单例不产生任何效果——这是「配置改了却不生效」最常见的一类解释。

**今天怎么做**（延伸，非语料原文）：现代工程里绝大多数 `BeanDefinition` 由组件扫描与自动配置生成，条件注解让「这份说明书要不要登记」在启动期被动态裁决；Spring 6 / Boot 3 的 AOT 路线甚至把一部分说明书在构建期就生成好。入口从 XML 变成了条件化注册，但「先有说明书、后有实例」这条结构没有变。

<div class="chapend" markdown="1">

**容易误解的地方**：
1. 「`@Component` 就是 Bean 本身」——它是登记说明书的依据，Bean 是之后按说明书造出来的对象。
2. 「`parent` 是 Java 继承」——它是 Bean 定义之间的配置继承，靠阶段 4 的合并实现，与类的 `extends` 无关。
3. 「`@Lazy` 对 prototype 也有意义」——语料明确 `lazyInit` 只对单例有效，prototype 本来就取用时才造。
4. 「多个实现时 `@Autowired` 会随便挑一个」——候选集合受 `autowireCandidate`、`primary`、`@Qualifier` 共同约束，冲突时会直接报错而不是静默选一个。
5. 「启动后改 `BeanDefinition` 能改变已有单例」——只能影响尚未创建的实例，已入一级缓存的对象不会被重建。

**记住这一句**：四种配置入口最终都收敛成一份 `BeanDefinition`，它在阶段 6 之前可写、在阶段 12 之后只读；能不能改配置，取决于对象有没有被造出来。

</div>
