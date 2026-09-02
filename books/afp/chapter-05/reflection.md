# Chapter5 Define Logic with Predicates｜思辨作答

## 1. Predicate 支持 `and / or / not` 组合业务规则，和 DMMF 纯业务函数组合的思想一致

Predicate 本质是**单入参返回布尔值的纯函数**，可以通过算子自由拼接成更复杂的规则。
DMMF 的核心主张：业务逻辑不写成过程式`if‑else`，而是把每一条独立约束定义成独立函数，再通过函数拼装表达复合业务规则。二者底层理念完全对齐：

1. 最小规则原子化：每一条业务约束独立定义，可单独单元测试；
2. 组合优先：复杂逻辑由小规则拼装而成，而非嵌套分支；
3. 无副作用：判断逻辑只读数据，不修改状态；
4. 声明式表达：只描述业务约束「是什么」，不描述判断执行步骤「怎么做」。

区别细节：DMMF 更偏向**类型层面杜绝非法状态**；Predicate 是运行时布尔判定，适合业务动态规则校验。

---

## 2. Java `Predicate.and()` / `.or()` 和 Elixir 可组合断言抽象的区别；适合 Command 校验还是 Domain 业务规则？

### Java `java.util.function.Predicate`

1. JDK 内置接口，提供`and() / or() / negate()`链式组合；
2. 限制：返回值只能是布尔值，**无法携带校验失败的错误描述信息**；短路求值（一旦`false`直接终止判断）；
3. 典型场景：**外层 Command 输入校验、简单过滤逻辑**。

### Elixir 可组合断言实现方式

Elixir 没有内置 Predicate 类型，使用普通函数 + 自定义高阶组合函数实现：

```
and_pred = fn p1, p2 -> fn x -> p1.(x) and p2.(x) end end
```

灵活性更强：我们可以不返回单纯布尔值，返回`ok / error`结构，实现**错误信息累积**（Validation 语义）。既可以做短路判断，也可以做全部校验收集全部报错。

### 分层适用边界（六边形分层原则）

1. **Command 层（应用入口）**：JDK Predicate 足够，完成语法、格式、基础非空校验；快速拒绝非法入参；
2. **Domain 领域层**：不推荐原生 JDK Predicate。领域规则需要**返回完整业务错误、支持错误聚合**，需要我们自定义 Specification/Validation 类型，对应 Vavr `Validation`。

> 
> 一句话结论：JDK Predicate 适合外层轻量校验；Domain 业务规则需要更丰富的返回模型，Elixir 自定义函数给了我们灵活扩展的原型思路。

---

## 3. Predicate 组合 ≈ DDD Specification 模式。对比两种写法的可读性

### Predicate 风格（Java 原生）

```
Predicate<Order> validAmount = o -> o.getAmount().compareTo(BigDecimal.ZERO) > 0;
Predicate<Order> validStatus = o -> o.getStatus() == DRAFT;
Predicate<Order> canSubmit = validAmount.and(validStatus);
```

优点：语法简洁，JDK 原生无需额外类；
缺点：规则没有专属命名对象，**无法携带业务错误文案、无法持久化序列化，语义弱**。

### Specification 完整 DDD 对象风格

```
public interface Specification<T> {
    ValidationResult test(T t);
    Specification<T> and(Specification<T> other);
}
Specification<Order> validAmount = new OrderPositiveAmountSpec();
Specification<Order> canSubmit = validAmount.and(draftStatusSpec);
```

优点：

- 独立领域类，拥有明确业务名称；
- 校验失败返回完整业务错误描述；
- 可扩展、可持久化、外部配置化（对接后续策略引擎 / DSL）；
缺点：代码量更多，需要定义一套类型体系。

### 可读性对比总结

- 简单一次性过滤逻辑：原生 Predicate 可读性更好；
- **长期迭代的核心领域业务规则**：Specification 模式可读性、可维护性更强，业务语义显性化，更适配 DDD 六边形架构。

> 
> Predicate 是 Specification 的**极简布尔子集**；Specification 是 Predicate 在领域工程上的增强版本。

---

## 4. 思考：是否可以用可组合 Predicate 重构你 UseCase 内部的业务 if‑else 判断？利弊是什么

### ✅ 收益（优点）

1. **规则原子化**：每一条判断独立抽离，支持单独单元测试；
2. **声明式代码**：UseCase 主流程不再充斥多层嵌套`if‑else`，业务逻辑阅读更直观；
3. **规则可复用**：同一条约束可以在多个 UseCase 中直接复用；
4. **规则动态编排**：运行时自由组合，为未来配置化策略引擎埋下扩展点。

### ❌ 弊端（代价）

1. JDK 原生 Predicate 只能返回布尔值，丢失**业务错误消息**，需要额外维护提示文案；
2. 过度拆分会产生大量细碎小规则，项目类数量膨胀；
3. 对于顺序强依赖、带副作用的业务流程，Predicate 并不适合；它只适合**纯数据判定规则**；
4. 简单一两次使用的临时判断，重构反而增加开发成本（过度设计）。

### 落地决策原则（工程权衡）

1. 如果是**多条可复用的核心业务约束**：推荐重构为可组合 Specification（增强版 Predicate）；
2. 如果是一次性、简短、局部流程判断：保留简单`if‑else`，保持代码简洁；
3. UseCase 职责：**业务规则交给 Domain 层 Specification，UseCase 负责流程编排、协调依赖、执行命令**，不把复杂业务判断写死在 UseCase 内部。

---

# 本章个人落地思考（复盘笔记）

1. Predicate 是可组合业务规则最简单的原型实现，是 Specification 模式的基础；
2. 分层策略：Command 层使用轻量 Predicate 做基础校验；Domain 层使用支持错误聚合的自定义 Specification；
3. 避免教条重构：不是所有`if‑else`都要消除；区分「可复用业务规则」和「一次性流程分支」；
4. 这套可组合布尔规则模型，正好是后续权限策略引擎、内部 DSL 的底层语义基础。
