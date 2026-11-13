# 第9章 AOP：四件套与两种代理

**一句话**：Spring AOP 把「在哪些方法上（Pointcut）执行什么增强（Advice）」组合成 Advisor/Aspect，最终靠 JDK 动态代理或 CGLIB 生成一个包着目标对象的代理实例。

**为什么需要它**：事务、日志、权限、耗时统计这些横切逻辑若写进业务类，每个方法都要重复一遍。语料用一整组专题（代理详解、JDK 动态代理和 CGLIB 代理、AOP 概念详解、AOP 核心源码、ProxyFactoryBean、通知顺序）把这件事拆成「概念 → 创建代理 → 调用代理 → 自动化」四层。更重要的是：理解代理，才能理解第 10 章的事务为什么「必须经由代理调用才生效」。

**核心结构**（语料给出的 AOP 四件套 + 代理三件套）：

| 概念 | 语料里的解释 | 一句话定位 |
|---|---|---|
| 连接点 JoinPoint | 方法执行等可插入增强的位置 | 「可以插在哪里」 |
| 通知 Advice | 指定方法的什么地方、执行什么操作 | 「插什么」 |
| 切入点 Pointcut | 指定通知用到哪些类的哪些方法 | 「插到哪些目标」 |
| 顾问 Advisor | `Pointcut` 与 `Advice` 的组合（也称通知器） | 把上面两者绑起来 |
| 切面 Aspect | 通知与切入点的组合 | 面向使用者的整体描述 |
| `AdvisedSupport` | 代理参数配置类 | 存 target、接口、通知等配置 |
| `AopProxyFactory` | `DefaultAopProxyFactory` | 决定用哪种代理 |
| `AopProxy` | `JdkDynamicAopProxy` / `ObjenesisCglibAopProxy` | 真正产出代理对象 |

**关键设计**：**创建代理被拆成固定三步**——语料的源码注释写得很明确：① 创建代理所需参数配置（通过 `AdvisedSupport`），② 根据代理参数获取 `AopProxy` 对象（`AopProxyFactory`），③ 通过 `AopProxy` 的 `getProxy()` 得到代理对象。这三步把「配置」「决策」「产出」分开，所以换代理方式不需要改业务代码。

```text
【代理怎么被造出来、怎么被调用】
 目标对象 target + 增强功能 Advice + 接口/类信息
   │ ① AdvisedSupport（含 ProxyConfig 的 proxyTargetClass）
   ▼
 ② DefaultAopProxyFactory.createAopProxy
   │    → JdkDynamicAopProxy 或 ObjenesisCglibAopProxy
   ▼
 ③ AopProxy.getProxy() → 代理对象放进容器（第6章初始化后）
   │
   ▼  业务调用代理方法
 拦截器链按 Advisor 顺序执行通知 → 再调用 target 方法
```

**拆解**：

### 四件套怎么区分

语料的定义很朴素：通知里有两个信息——「方法的什么地方」（之前、之后、包裹目标方法、抛异常后）和「执行什么操作」；切入点负责圈定「哪些类的哪些方法」；`Advisor` 把两者绑定，因为增强逻辑必须与作用位置组合才能生效；`Aspect` 则是从使用者视角对「通知 + 切入点」的统称。语料还列出了通知类型相关的类与接口，案例输出里能看到 `@Around`、`@Before`、`@After`、`@AfterReturning`、`@AfterThrowing` 的执行顺序。**区分它们的意义在于排查「切面不生效」：先看切入点有没有匹配，再看 Advisor 顺序，最后才怀疑代理方式**。

### 什么时候用 JDK 代理、什么时候用 CGLIB

语料给出 `DefaultAopProxyFactory` 的判定条件：`optimize = true`、`proxyTargetClass = true`、或「未指定代理接口」时创建 CGLIB 代理；否则走 JDK 动态代理。产出分别是 `JdkDynamicAopProxy` 与 `ObjenesisCglibAopProxy`。`@EnableAspectJAutoProxy` 的 `proxyTargetClass` 属性正是这个开关：语料在事务专题里也写过，设为 `true` 就是强制使用 CGLIB 创建代理。由此还引出一条实践结论：只按接口注入时，JDK 代理够用；需要按具体类注入、或目标类没有接口时，就得让 CGLIB 上场。

### 手动创建与自动创建代理

语料把创建方式分成两大类：**手动方式**需要硬编码一个个创建代理（如 `ProxyFactory`、`AspectJProxyFactory`、`ProxyFactoryBean`），其中 `ProxyFactoryBean` 通过实现 `FactoryBean` 给指定 Bean 创建代理，创建代理的三个关键信息是 Advice、target、proxy；**自动化方式**用于 Spring 环境，靠 Bean 后置处理器批量为符合条件的 Bean 创建代理，代表就是第 6 章提到的 `AnnotationAwareAspectJAutoProxyCreator`。两条路最终都走 `ProxyFactory` / `AdvisedSupport` 这一套，区别只在「谁发起」。

### 代理带来的两个固有限制

其一是自身调用：`this.m2()` 走的是目标对象内部调用，不经过代理的拦截器链，所以加在 `m2` 上的增强不会执行；语料在事务失效专题里把它列为第 4 种情况，并给出「注入自己再通过代理调用」和 `AopContext.currentProxy()`（配合 `exposeProxy = true`）两类解法。其二是对象身份：`@PostConstruct` 里的 `this`、以及直接按具体类型做判断时，你面对的可能不是代理。语料注释把 `exposeProxy` 解释为「是否需要将代理对象暴露在 ThreadLocal 中，为 true 时可以通过 `AopContext#currentProxy` 获取当前代理对象」——这句话本身就说明了「目标对象」与「代理对象」是两个实例。

**今天怎么做**（延伸，非语料原文）：现代 Spring Boot 默认对 AOP 采用 CGLIB 代理（通过配置属性控制），并更强调按接口编程以减少对具体类的依赖；AspectJ 的编译期/加载期织入在需要更细粒度增强时仍是可选项。Spring 6 / Boot 3 对代理生成与 AOT 的兼容做过调整，但「自身调用绕过代理」「最终暴露的是代理对象」这两条限制依旧成立，也是线上事务问题最常见的根因。

<div class="chapend" markdown="1">

**容易误解的地方**：
1. 「AOP 是修改了目标类的字节码」——语料的 Spring AOP 主线是用代理对象包裹目标对象，通过拦截器链增强，而不是改写原类。
2. 「Advisor 和 Aspect 是一回事」——语料把 Advisor 定义为 `Pointcut + Advice` 的具体组合，Aspect 是通知与切入点组合的整体概念。
3. 「`@EnableAspectJAutoProxy` 只是个开关，不影响类型」——`proxyTargetClass` 会决定用 CGLIB 还是 JDK 代理，进而影响能否按具体类注入。
4. 「同类内部调用也能触发通知」——`this` 不是代理，拦截器链不会被执行。
5. 「`ProxyFactoryBean` 是容器里创建代理的唯一入口」——语料明确区分手动与自动化两类，容器里主要是后置处理器批量创建。

**记住这一句**：Spring AOP 把 `Pointcut + Advice` 绑成 Advisor，再用 `JdkDynamicAopProxy` 或 `ObjenesisCglibAopProxy` 生成一个代理对象替换掉原始 Bean；因此「有没有经过代理」是判断一切增强是否生效的第一问。

</div>
