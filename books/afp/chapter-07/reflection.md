# Chapter7 Access Shared Environment with Reader（Reader Monad）｜思辨完整答案

## 1. Reader Monad 本质是函数式延迟依赖注入，不依赖 Spring 容器

### Reader 底层本质

`Reader<E, A` 本质就是一个**纯函数：`env → value`**
它本身不持有环境，只是**保存一段接收环境才能执行的计算逻辑**：

- **延迟绑定依赖**：定义业务管道的时候，不需要立刻传入外部依赖；直到最终运行时刻，才把完整环境注入进去求值。
- 无全局容器、无反射、无注解；全部靠高阶函数组合完成依赖传递。

对比 Spring DI：

- Spring：启动期容器完成对象装配，运行时从上下文直接获取 Bean；是**运行时对象注入**
- Reader：先组合业务计算流水线，最后传入环境执行；是**函数式延迟注入**

核心语义：业务逻辑描述**不耦合具体实现**，环境可以在最后一刻替换。

伪代码示意：

```
// Reader = 一个接收环境的函数
Reader<Env, Result> workflow = env -> businessLogic(env.repository, env.config);
// 最后执行时才传入真实环境 / 测试环境
Result real = workflow.run(productionEnv);
Result test = workflow.run(testEnv);
```

## 2. OOP 六边形依靠接口 + Spring DI；FP 依靠高阶函数 / Reader 注入输出端口，两套依赖注入方案各自取舍

### 方案 A｜OOP 六边形（Spring + 端口接口）

**实现方式**
UseCase 构造函数接收输出端口接口（`OrderRepository`），Spring 自动注入实现类。

```
@Service
public class CreateOrderUseCase {
    private final OrderRepository repo;
    public CreateOrderUseCase(OrderRepository repo){ this.repo = repo; }
}
```

✅优点

1. 符合 Java 生态习惯，团队上手成本低；
2. Spring 完整管理对象生命周期、事务、AOP；
3. 适合大量基础设施 Bean、长生命周期资源（连接池）。
❌缺点
4. 依赖在**对象构造阶段绑定**；运行时不方便动态切换整套环境；
5. 单元测试需要 Mock 工具（Mockito）替换实现；
6. 业务链路组合和容器生命周期耦合在一起。

### 方案 B｜FP Reader / 高阶函数注入

**实现方式**
UseCase 返回一个`Reader<Environment, Result>`，环境里面携带所有输出端口、配置；业务流程自由链式组合，**执行前随时替换整套环境**。
✅优点

1. 依赖延迟注入，同一个业务管道可以复用在生产 / 测试两种环境；
2. 纯函数组合，不需要反射；测试直接构造测试环境对象即可；
3. 多条业务流水线可以自由拼接、复用。
❌缺点
4. Java 没有原生 Reader 类型类，需要手动封装；样板代码增加；
5. 无法天然托管资源生命周期（数据库连接池、线程池）；
6. 大型系统全部用 Reader 管理依赖会变得笨重。

### 取舍结论（Tom 混合范式）

> 
> **长生命周期基础设施交给 Spring DI；单次请求上下文（配置、审计用户、临时端口实现）适合 Reader 模式。两者共存，不是二选一。**

## 3. 单元测试场景：Reader 如何实现无 mock 框架的依赖替换？对比 Mockito 方案

### Mockito 传统方案

1. 在测试代码创建接口 Mock 对象；
2. 通过依赖注入把 Mock 交给 UseCase；
3. 使用 when‑thenReturn 打桩模拟返回值；
4. 依赖第三方库、字节码代理。

### Reader 无 Mock 测试方案

Reader 的环境本身就是一个普通 POJO：

- **生产环境实例：注入真实 Repository 实现**
- **测试环境实例：注入手写的内存实现（InMemoryRepository）**

**不需要任何 Mock 框架**：同一个 Reader 业务流程，执行时传入不同环境对象即可。

```
// 业务管道只定义一次
Reader<AppEnv, Result> workflow = createOrderPipeline();

// 生产运行
workflow.run(productionEnv);
// 单元测试：直接传入内存实现的环境对象
workflow.run(testEnv);
```

✅Reader 测试优势

- 测试替身是**真实手写实现**，不是动态代理；行为更加可预测；
- 消除 Mock 打桩带来的脆弱测试；
❌局限
对于大量细粒度外部协作对象，手写内存替身成本更高，这种场景 Mockito 依然高效。

对比总结：

- Reader：**整套环境整体替换，适合端到端业务规则测试**
- Mockito：**单个依赖精细打桩，适合隔离微小协作对象**

## 4. 决策清单：哪些外部依赖适合 Reader 注入，哪些交给 Spring 容器管理（Java 混合范式）

### ✅适合用 Reader Monad 托管的依赖（单次请求 / 计算上下文）

1. 请求级上下文：当前登录用户 ID、租户编号、审计信息
2. 动态业务配置：本次流程使用的业务参数、策略开关
3. 可替换的业务端口实现（内存实现 / 真实实现），用于业务规则测试
4. 一组需要整体切换的协作依赖（整套测试环境）

> 
> 特点：生命周期短，每次执行可以完全独立替换，不需要管理资源。

### ✅适合交给 Spring 容器管理的依赖（全局长生命周期基础设施）

1. 数据库 Repository 真实实现（连接池由 Spring 管理）
2. 消息队列客户端、HTTP 外部服务客户端（资源需要初始化 / 销毁）
3. 事务管理器、AOP 切面、全局拦截器
4. 固定单例的基础设施组件

> 
> 特点：应用启动创建一次，整个进程复用，需要生命周期管理。

### 混合架构落地策略（六边形 OOP+FP）

1. **Spring 负责创建所有基础设施 Bean**；
2. UseCase 内部组装 Reader，Reader 的环境对象持有 Spring 注入的端口引用；
3. 运行阶段：业务逻辑用 Reader 延迟组合；测试阶段：环境替换为内存实现；
4. 领域层保持纯值，不感知任何一种注入方式。

# 本章精炼笔记（写入 reflection.livemd）

1. Reader = `env → value`，本质是**延迟求值的函数式依赖注入**，运行前随时切换整套执行环境；
2. Spring DI 管理全局长生命周期资源；Reader 管理单次计算 / 请求的上下文依赖；二者互补而非替代；
3. Reader 测试不需要 Mock 框架，通过替换整个环境对象实现业务替身；Mockito 适合细粒度隔离；
4. 混合范式最佳实践：基础设施交给 Spring，动态请求上下文使用 Reader 进行函数式编排。
