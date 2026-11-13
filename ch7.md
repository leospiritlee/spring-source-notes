# 第7章 三级缓存与循环依赖：三个 Map 怎么把死结解开

**一句话**：Spring 用三张表分别存「成品」「早期对象」「能产出早期引用的工厂」，把「对象还没造完就先给个引用」变成合法操作，从而解开 setter 方式的单例循环依赖。

**为什么需要它**：A 依赖 B、B 依赖 A，按普通思路两边都会卡在「等对方先造完」。语料专门用两篇（第 28 篇 Bean 循环依赖详解、第 56 篇「循环依赖不用三级缓存可以么？」）推演这件事，并给出结论：**不能用两级缓存替代三级缓存**。原因不是「多一层更保险」，而是第三级保存的是工厂，只有在真的被提前取用时才决定「给原始对象还是给代理」，这个决定无法在造壳时提前做。

**核心结构**（`DefaultSingletonBeanRegistry` 里的三张表 + 一张名单）：

| 级 | 语料里的声明 | 存什么 |
|---|---|---|
| 第 1 级 | `Map<String, Object> singletonObjects` | 完全创建好的单例 Bean，`beanName → 实例` |
| 第 2 级 | `Map<String, Object> earlySingletonObjects` | 早期的 Bean，`beanName → 实例` |
| 第 3 级 | `Map<String, ObjectFactory<?>> singletonFactories` | 单例 Bean 的 `ObjectFactory`，`beanName → 工厂` |
| 名单 | `Set<String> singletonsCurrentlyInCreation` | 正在创建中的 Bean 名称，创建完移除 |

**关键设计**：**第三级存的是「怎么拿到早期引用」这个过程，不是引用本身**。语料源码注释写着 `addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))`，并注明 `getEarlyBeanReference` 内部「可以在这个里面通过 aop 生成代理对象」。也就是说，代理要不要生成、生成成什么样，要等到别人真的来取时才定。

```text
【A → B → A 的 setter 循环依赖】
 getBean(A)
   │ 造壳 A → addSingletonFactory(A, ObjectFactory)  [3级]
   ▼
 populateBean(A) 需要 B → getBean(B)
   │ 造壳 B → addSingletonFactory(B)               [3级]
   ▼
 populateBean(B) 需要 A → getSingleton(A, false)
   │ 命中 3级 → 调 getEarlyBeanReference → 得早期引用
   │ 早期引用移入 2级 earlySingletonObjects
   ▼
 B 填充完成 → 初始化 → 入 1级 singletonObjects
   │
   ▼
 A 拿到成品 B → 填充完成 → 初始化 → 入 1级
```

**拆解**：

### 三张表各自解决什么问题

第 1 级是最终答案：`getBean` 一开始就先查它，命中直接返回。第 2 级是「已经被别人取用过的早期引用」，避免同一个 Bean 每次被提前取用都重新生成一份。第 3 级是尚未兑现的工厂，它把「提前暴露」这件事推迟到真正需要的那一刻。语料把 `getSingleton(beanName, false)` 的行为写得很清楚：第二个参数为 `false` 时只查第 1、2 级；而「若 Bean 存在三级缓存中且 `beanName` 在当前创建列表」，调用该方法时才会把 Bean 从三级缓存移到二级缓存。**三级到二级不是自动升级，而是被取用时触发的兑现**。

### 为什么两级不够用

如果只有「成品表 + 早期对象表」，容器就必须在造壳完成那一刻决定早期引用是什么。可这个对象后面还可能被 `BeanPostProcessor` 换成代理（第 6 章）：给了原始对象，别人注入的就是没增强的版本；想给代理，又得在属性填充之前就把初始化后阶段的工作提前跑一遍。语料第 56 篇把答案直接写成「不可用」，而源码里的解法就是把这段决定封装进 `ObjectFactory`，由 `getEarlyBeanReference` 在取用时代理与原始对象之间做选择。**第三级买的不是冗余，是「延迟决定」**。

### 为什么还要核对 exposedObject

语料在创建流程尾部有一段容易被跳过的判断：初始化得到的是 `exposedObject`，随后再 `getSingleton(beanName, false)` 取 `earlySingletonReference` 做比对。原因是若 A 的早期引用已经被 B 拿走，而初始化后又生成了另一个代理对象，容器里就会出现「B 手里的 A」与「池子里的 A」不一致。语料注释解释了二级缓存里出现该 Bean 意味着什么（它确实被提前取用过），这段代码就是在为这种情形兜底。读懂它，才能明白循环依赖与 AOP 不是两个独立话题，而是同一处时序问题。

### 哪些循环依赖解不开

语料把注入方式分成构造器与 setter 两种，并强调「本文未指明 bean scope 默认情况下，所有 bean 都是单例的，即下面所有问题都是在单例的情况下分析的」。构造器注入在造壳时就需要对方，此时还没有壳可以放进三级缓存，因此无解；`singletonsCurrentlyInCreation` 发现同名 Bean 又被引回来，便抛出语料里提到的 `BeanCurrentlyInCreationException`。prototype 也不在这套缓存账里——三级缓存是为单例准备的。此外语料特别提醒「需注意循环依赖注入的是半成品」：解开的是创建顺序，不是「注入的对象已经完成初始化」。

**今天怎么做**（延伸，非语料原文）：后来 Spring Boot 的默认配置对循环依赖的态度收紧，一些版本会直接禁止通过早期引用来兜底；构造器注入 + 显式重新分层成为更被推荐的做法，因为它把隐式的「半成品被别人持有」变成启动时就能发现的依赖环。这些属于策略与默认值的变化；语料里 5.2.x 的三级缓存机制依然是理解这些报错的前提。

<div class="chapend" markdown="1">

**容易误解的地方**：
1. 「三级缓存是为了性能」——它是为了解决「提前暴露时该给原始对象还是代理」，与缓存加速无关。
2. 「二级缓存也够用」——语料第 56 篇的结论是不可用，因为两级缓存必须在造壳时就把早期引用定死。
3. 「三级缓存会自动升到二级」——只有被 `getSingleton(beanName, false)` 取用、且该 Bean 正在创建中时才发生转移。
4. 「循环依赖被解开说明注入的是成品」——注入的是半成品/早期引用，字段与增强都可能还没齐。
5. 「所有循环依赖 Spring 都能处理」——构造器方式、非单例作用域都不在这套机制的能力范围内。

**记住这一句**：三级缓存用「工厂」把早期引用的决定推迟到被取用那一刻，因此既能解开 setter 方式的单例循环依赖，又不会绕过 AOP 代理；解不开的那些，都是因为造壳时根本没有可提前暴露的对象。

</div>
