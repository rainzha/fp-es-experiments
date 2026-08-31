# Advanced Functional Programming with Elixir｜阅读思辨清单
> 适配背景：已读完 DMMF + Clean Applications with Hexagonal Architecture (1‑6章)
> 最终生产技术栈：Java26 + Spring Boot + Vavr 0.11
> 使用规范：
> 1. 阅读章节前，浏览本章全部问题，带着目标阅读
> 2. 产出两份笔记：`chapter‑XX/exercises.livemd`（书本代码练习）｜`chapter‑XX/reflection.livemd`（回答下面思辨题，跨语言对比思考）
> 3. 完成一章独立提交 Git

## 通用固定3问（每一章必读）
1. 这个抽象如果用 F#（DMMF静态类型）实现是什么写法？编译器带来哪些编译期约束？
2. 如果移植到 Java26 + Sealed Record + Vavr0.11，该如何落地？哪些特性 Java 无法原生实现，需要折中？
3. 这个模式归属六边形架构哪一层（Domain / Application / Adapter）？事件溯源场景是否可以复用这套思路？

---

## Chapter 1｜Build FunPark: Model Real‑World Data
1. DMMF从一开始就用F#代数类型锁定合法状态；Elixir依靠`struct + 构造函数约束`建模。动态语言如何规避非法领域对象？
2. 对比Java值对象（Record + 私有构造 + 静态工厂）和Elixir Struct建模风格的异同。
3. DDD通用语言(Ubiquitous Language)在FunPark案例中如何体现？和你Java项目里`Command / UseCase / ValueObject`命名规范对照。
4. 本章没有复杂高阶抽象，为什么FP建模依然优先「数据优先」而不是行为优先？对比传统OOP实体。

## Chapter 2｜Implement Domain‑Specific Equality with Protocols（Eq）
1. DMMF/F#默认结构性相等；Elixir通过Protocol实现**业务上下文相等**。DDD什么时候不能直接使用原生相等判断？举业务案例。
2. Java Record默认的equals()，什么时候需要重写自定义业务相等规则？和本章Eq协议设计思路对齐。
3. 领域唯一性校验（例如Slug唯一），可插拔Eq协议能给业务校验带来什么灵活性？
4. 集合去重、匹配判断场景，Eq抽象在Domain层如何减少重复的if‑else判断？

## Chapter 3｜Create Flexible Ordering with Protocols（Ord）
1. Ord = 业务自定义比较逻辑。对比F#、Java `Comparator`三者在**可组合性**上的差异。
2. 事件溯源查询场景：对历史事件按业务规则排序，Ord协议的价值是什么？
3. 在你的Java六边形项目中，哪些领域对象需要封装独立的业务排序规则？
4. 协议模式支持**多种排序策略共存**，对比策略模式（OOP）两种方案取舍。

## Chapter 4｜Combine with Monoids（幺半群）
1. Monoid的核心：**存在空初始值，并且值可以安全合并**。解释事件溯源`fold折叠事件流`就是典型Monoid场景。
2. DMMF没有专门讲解Monoid；F#如何手写实现相同的规约合并逻辑？
3. Java `Stream.reduce` / Vavr fold如何等价实现Monoid；和Elixir实现对比。
4. 识别你自己的错误体系：`ValidationErrors`多条错误聚合，是不是Monoid？解释为什么是/不是。

## Chapter 5｜Define Logic with Predicates（可组合断言）
1. Predicate支持`and/or/not`组合业务规则，和DMMF纯业务函数组合的思想一致。
2. Java `Predicate.and()` / `.or()` 和Elixir可组合断言抽象的区别；适合Command校验还是Domain业务规则？
3. Predicate组合 ≈ DDD Specification模式。对比两种写法的可读性。
4. 思考：是否可以用可组合Predicate重构你UseCase内部的业务if‑else判断？利弊是什么。

## Chapter 6｜Compose in Context with Monads
1. Monad解决嵌套上下文链式调用（`bind/flatMap`）。对比Vavr Either.flatMap、DMMF Result绑定逻辑。
2. Elixir依靠第三方库协议模拟Monad，没有静态检查；Java只能手动flatMap，权衡两者优缺点。
3. UseCase完整业务链路编排：FP Monad管道 vs OOP分步调用输出端口，两种编排风格对比。
4. 完整流程：Command → Validate → Build Domain Event → Persist Event，整条链路如何用Monad串联。

## Chapter 7｜Access Shared Environment with Reader（Reader Monad）
1. Reader Monad本质是**函数式延迟依赖注入**，不依赖Spring容器。
2. OOP六边形依靠接口 + Spring DI；FP依靠高阶函数/Reader注入输出端口，两套依赖注入方案各自取舍？
3. 单元测试场景：Reader如何实现无mock框架的依赖替换？对比Mockito方案。
4. 整理决策清单：哪些外部依赖适合Reader注入，哪些交给Spring容器管理（Java混合范式）。

## Chapter 8｜Manage Absence with Maybe
1. Maybe/Option用来表达**值缺失**；区分Vavr Option、F# Option语义。
2. 严格区分语义边界：Maybe代表「可选值」；Either代表「业务失败」。对照你分层校验规范。
3. 数据库查询返回空结果（Adapter层），使用Maybe包装返回值；Domain层是否感知空值？
4. 什么时候在Java项目直接返回null，什么时候强制使用Vavr Option？制定工程规范。

## Chapter 9｜Model Outcomes with Either
1. `Either<Error,Value>`承载业务失败；横向对比：DMMF Result、Vavr Either / Validation三者语义细微区别。
2. Elixir `{:ok, val}` / `{:error, reason}`运行时元组 vs F#静态编译Result，动态/静态方案的取舍。
3. 对照你六边形分层校验：Command格式校验用Validation，Domain业务规则校验用Either，这套规则如何映射成Elixir代码？
4. 定义取舍规则：**什么时候不用Either，直接抛出异常**，复用到你的Java OOP+FP混合项目。

## Chapter 10｜Coordinate Tasks with Effect
1. Effect = **延迟执行的副作用**：先描述IO动作，事务提交之后统一执行。和你实现的「延迟领域事件」思想同源。
2. DMMF只简单说明IO放在外层；Effect把副作用变成可组合的值。对比Java延迟事件 / 发件箱模式。
3. 严格分层边界：Domain（无Effect纯逻辑）、Application编排Effect、Adapter真正执行Effect。
4. 事件溯源完整流程：先构造Effect列表，事务成功后统一持久化 + 发布事件。这套思路如何反向优化你的Java UseCase实现？

