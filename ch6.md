# 第6章 初始化与 BeanPostProcessor：代理在这里换壳

**一句话**：`initializeBean` 按「Aware 回调 → 初始化前 → 初始化方法 → 初始化后」跑完四段，其中最后一段的 `BeanPostProcessor` 可以把原始对象整个换成代理。

**为什么需要它**：这是全书最容易被忽略、却解释了最多「不生效」的一步。事务与 AOP 之所以能拦截你的方法，是因为容器最终放进单例池的不是你写的那个对象，而是后置处理器在这一步返回的代理。语料在讲 `AnnotationAwareAspectJAutoProxyCreator` 时明确说过：它是 `BeanPostProcessor` 类型，可以在 Bean 生命周期中对 Bean 进行操作，比如生成代理，源码可以从 `postProcessAfterInitialization` 看起。

**核心结构**：

| 小阶段 | 语料中的名字 / 接口 | 典型动作 |
|---|---|---|
| Aware 回调 | `invokeAwareMethods`、`BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware` | 把容器信息回送给 Bean |
| 初始化前 | `postProcessBeforeInitialization` | 对实例做前置处理 |
| 初始化 | `InitializingBean` 的 `afterPropertiesSet`、`@PostConstruct`、定义 Bean 时指定的初始化方法 | 业务自己的初始化逻辑 |
| 初始化后 | `postProcessAfterInitialization` | 生成并返回代理对象 |
| 全部单例完成 | `SmartInitializingSingleton`、阶段 10 | 所有单例就绪后的收尾 |

**关键设计**：**后置处理器返回什么，容器最终就存什么**。`BeanPostProcessor` 的两个方法签名都以 `Object` 为返回值，语料里能看到 `postProcessAfterInitialization` 的默认实现是 `return bean`——正因为可以返回别的对象，代理替换才成立。

```text
【initializeBean 的四段】
 半成品（属性已填）
   │ ① invokeAwareMethods：BeanName / ClassLoader / BeanFactory
   ▼
 ② BeanPostProcessor.postProcessBeforeInitialization
   ▼
 ③ 初始化：afterPropertiesSet → @PostConstruct → 指定初始化方法
   │      此刻的 this 仍是原始对象，不是代理
   ▼
 ④ BeanPostProcessor.postProcessAfterInitialization
   │      AnnotationAwareAspectJAutoProxyCreator 在这里返回代理
   ▼
 exposedObject → 一级缓存 singletonObjects
```

**拆解**：

### Aware 回调把容器信息递给 Bean

语料给出 `invokeAwareMethods` 的源码：先判断 Bean 是否实现了 `Aware`，再依次处理 `BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`。这一步的意义是让 Bean 能拿到自己的名字、类加载器与所属工厂，而不必自己去 `new` 或静态查找容器。它排在初始化方法之前，因此初始化逻辑里已经可以使用这些回调得到的信息。

### 初始化方法其实有三条来源

语料在「Bean 初始化阶段」里明确列了两步：调用 `InitializingBean` 接口的 `afterPropertiesSet`，再调用定义 Bean 时指定的初始化方法；并列举了指定初始化方法的三种方式（XML、`@Bean`、API）。此外 `@PostConstruct` 也在语料的案例输出里出现过（`postConstruct1()`、`postConstruct2()`）。所以「初始化」不是一个注解，而是一组按时序执行的回调集合：**它们都发生在属性填充之后，此时字段才齐全**。

### 初始化后为什么会换一个对象

`postProcessAfterInitialization` 的默认实现只是 `return bean`，但实现类可以返回别的东西。语料在 AOP 专题里点明 `AnnotationAwareAspectJAutoProxyCreator` 就是靠这个位置为符合条件的 Bean 自动生成代理对象；同一份语料讲 `@EnableAspectJAutoProxy` 时还提到 `exposeProxy` 与 `AopContext.currentProxy()`，用来在需要时拿到当前代理对象——这从侧面印证了「业务拿到的实例」与「原始目标对象」是两个东西。第 9、10 章的切面与事务，全靠这一步落地。

### 阶段 10 与 SmartInitializingSingleton

语料的生命周期清单把「所有单例 bean 初始化完成后阶段」单列为阶段 10，对应 `SmartInitializingSingleton`。它和 `BeanPostProcessor` 的区别是作用域：BPP 对每个 Bean 各跑一次，而这个回调在所有单例都就绪之后才跑，适合做「需要看到全局」的检查或预热。第 2 章的阶段 12 `finishBeanFactoryInitialization` 正是把这一整批单例造完的地方，两者是同一条时间线的不同刻度。

**今天怎么做**（延伸，非语料原文）：现代 Spring 里 `@PostConstruct` 已逐步被推荐用法替换或与其他初始化手段并用，构造完成后即做校验、把复杂初始化挪到专门的生命周期回调更常见；Spring 6 / Boot 3 对 AOT 与代理生成路径做了调整，但「后置处理器返回的对象才是最终 Bean」这条约定没变。顺带一提，想在初始化逻辑里调用本类的增强方法，依然要绕开 `this`，这是代理模型的固有限制而非版本差异。

<div class="chapend" markdown="1">

**容易误解的地方**：
1. 「`@PostConstruct` 里 `this` 就是容器对外暴露的那个 Bean」——代理在初始化后阶段才生成，两者可能不是同一对象。
2. 「`BeanPostProcessor` 只是通知回调」——它返回 `Object`，返回代理就是对实例的替换。
3. 「`afterPropertiesSet` 和初始化方法只能二选一」——语料明确是两步依次调用，指定初始化方法还有三种写法。
4. 「Aware 是依赖注入的一部分」——它发生在初始化阶段的 Aware 回调里，送进来的是容器自身的信息，不是业务 Bean。
5. 「所有单例造完后就没有容器回调了」——阶段 10 还有 `SmartInitializingSingleton`，关闭时还有销毁前与销毁两个阶段。

**记住这一句**：初始化的四段以「初始化后」收尾，而 `postProcessAfterInitialization` 可以返回另一个对象；AOP 与事务的代理就在这一步换壳，所以「你写的类」和「容器给你的 Bean」经常不是同一个实例。

</div>
