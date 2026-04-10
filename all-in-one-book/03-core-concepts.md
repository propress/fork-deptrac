# 第 3 章：核心概念深入理解

> 🟢 新手 → 🟡 熟悉
> 
> 阅读时间：约 15 分钟

---

## 3.1 概念全景图

Deptrac 的工作围绕 5 个核心概念展开：

```
┌─────────────────────────────────────────────────────────┐
│                   Deptrac 核心概念                        │
│                                                          │
│   ┌──────────┐     ┌───────────┐     ┌──────────┐       │
│   │  Token    │────▶│ Collector │────▶│  Layer   │       │
│   │ (类/函数) │     │ (收集器)   │     │  (层)    │       │
│   └──────────┘     └───────────┘     └────┬─────┘       │
│                                           │              │
│                                           ▼              │
│                                      ┌─────────┐        │
│                                      │ Ruleset │        │
│                                      │ (规则集) │        │
│                                      └────┬────┘        │
│                                           │              │
│                                           ▼              │
│                                     ┌──────────┐        │
│                                     │Violation │        │
│                                     │ (违规)    │        │
│                                     └──────────┘        │
└─────────────────────────────────────────────────────────┘
```

## 3.2 Token（令牌）

**Token** 是 Deptrac 分析的最小单位。它可以是：

| Token 类型 | 说明 | 例子 |
|-----------|------|------|
| **class-like** | 类、接口、Trait、Enum | `App\Entity\User` |
| **function** | 函数 | `App\Helper\formatDate` |
| **file** | 文件 | `src/config.php` |
| **superglobal** | PHP 超全局变量 | `$_POST`、`$_GET` |

### 依赖类型（Dependency Types）

Deptrac 可以检测多种类型的依赖关系：

```yaml
# 在配置中可以控制检测哪些依赖类型
analyser:
  types:
    - class              # 类级别依赖（new、类型提示、继承等）
    - use                # use 语句
    - file               # 文件级别依赖
    - function           # 函数级别依赖
    - function_call      # 函数调用
    - superglobal        # 超全局变量访问
```

```php
<?php
// Deptrac 能检测到的各种依赖方式：

use App\Service\Logger;          // ← use 依赖

class UserController
{
    public function __construct(
        private Logger $logger   // ← class 依赖 (类型提示)
    ) {}

    public function save()
    {
        $user = new User();      // ← class 依赖 (new 实例化)
        
        if ($user instanceof Admin) {  // ← class 依赖 (instanceof)
            // ...
        }
        
        $data = $_POST['name'];  // ← superglobal 依赖
        
        array_map(fn($x) => $x, []); // ← function_call 依赖
    }
}
```

## 3.3 Collector（收集器）

**Collector** 决定哪些 Token 属于哪个 Layer。把它想象成一个"过滤器"或"分拣员"。

```
         所有的 PHP 类/函数/文件
         ┌─────────────────────┐
         │ UserController      │
         │ ProductController   │
         │ UserService         │     Collector: classLike
         │ OrderService        │─────(value: .*Controller.*)────▶ Controller 层
         │ UserRepository      │
         │ OrderRepository     │     Collector: classLike
         │ User (Entity)       │─────(value: .*Service.*)───────▶ Service 层
         │ Order (Entity)      │
         │ helpers.php         │     Collector: classLike
         └─────────────────────┘─────(value: .*Repository.*)────▶ Repository 层
```

### 最常用的收集器一览

| 收集器 | 匹配依据 | 使用频率 | 典型场景 |
|--------|---------|---------|---------|
| `classLike` | 类名正则 | ⭐⭐⭐⭐⭐ | 大多数场景的首选 |
| `directory` | 文件路径正则 | ⭐⭐⭐⭐ | 按目录组织代码时 |
| `bool` | 组合多个收集器 | ⭐⭐⭐ | 复杂条件（AND/OR/NOT） |
| `implements` | 实现的接口 | ⭐⭐⭐ | 基于接口的架构 |
| `extends` | 继承的父类 | ⭐⭐ | 基于继承的架构 |
| `composer` | Composer 包 | ⭐⭐ | 分析第三方依赖 |
| `attribute` | PHP 8 属性 | ⭐⭐ | 使用属性标注的代码 |

### classLike 收集器详解

最常用的收集器，按类名正则匹配。匹配的是**全限定类名（FQCN）**。

```yaml
layers:
  - name: Controller
    collectors:
      - type: classLike
        value: .*Controller.*
```

> **内部实现**：Deptrac 会将你的 value 包裹为 `/.*Controller.*/i`，即大小写不敏感的正则。

```
你写的:     .*Controller.*
实际正则:   /.*Controller.*/i

匹配示例:
  ✅ App\Controller\UserController     ← 包含 "Controller"
  ✅ App\Http\Controllers\ApiController ← 包含 "Controller"
  ❌ App\Service\UserService           ← 不包含 "Controller"
```

### directory 收集器详解

按文件所在目录匹配，路径是相对于项目根目录的：

```yaml
layers:
  - name: Domain
    collectors:
      - type: directory
        value: src/Domain/.*
```

### bool 收集器：组合逻辑

当单个收集器不够用时，使用 `bool` 收集器组合：

```yaml
layers:
  - name: DomainService
    collectors:
      - type: bool
        must:                           # AND 条件：必须同时满足
          - type: classLike
            value: .*Service.*
          - type: directory
            value: src/Domain/.*
        must_not:                       # NOT 条件：必须不满足
          - type: classLike
            value: .*Abstract.*         # 排除抽象类
```

```
bool 收集器的逻辑：

   must[0] AND must[1] AND ... AND NOT must_not[0] AND NOT must_not[1] ...

   即：在 src/Domain/ 目录下，类名包含 Service，但不包含 Abstract
```

### 一个层可以有多个收集器

```yaml
layers:
  - name: Infrastructure
    collectors:
      - type: directory
        value: src/Infrastructure/.*
      - type: classLike
        value: .*Adapter.*        # 也收集名字包含 Adapter 的类
```

> 多个收集器之间的关系是 **OR**（满足任一即可）。

## 3.4 Layer（层）

**Layer** 是一组 Token 的逻辑分组。它代表你架构中的一个"层"或"模块"。

```
┌─────────── Controller 层 ───────────┐
│                                      │
│  UserController                      │
│  ProductController                   │
│  OrderController                     │
│  ...                                 │
│                                      │
│  (所有被 Collector 匹配到的类)        │
└──────────────────────────────────────┘
```

### 一个 Token 可以属于多个层

```yaml
layers:
  - name: Controller
    collectors:
      - type: classLike
        value: .*Controller.*
  - name: HttpLayer
    collectors:
      - type: directory
        value: src/Http/.*
```

如果 `src/Http/UserController.php` 存在，那么 `UserController` 同时属于 Controller 层和 HttpLayer 层。

### 私有层（Private Layer）

标记 `private: true` 的层，其中的类只能被同层的其他类引用：

```yaml
layers:
  - name: InternalHelpers
    collectors:
      - type: directory
        value: src/Internal/.*
    private: true                # 这个层的类只能被自己层内的类使用
```

## 3.5 Ruleset（规则集）

**Ruleset** 定义了层与层之间的依赖关系。

```yaml
ruleset:
  Controller:                  # 对于 Controller 层
    - Service                  # ✅ 允许依赖 Service
    - DTO                      # ✅ 允许依赖 DTO
  Service:
    - Repository               # ✅ 允许依赖 Repository
    - DTO                      # ✅ 允许依赖 DTO
  Repository:
    - Entity                   # ✅ 允许依赖 Entity
  Entity: ~                    # 不允许依赖任何层
  DTO: ~                       # 不允许依赖任何层
```

### 核心原则：默认禁止

```
                    显式声明的 ──── ✅ 允许
                   /
   层间依赖关系 ──<
                   \
                    未声明的 ────── ❌ 禁止（默认！）
```

### 用图可视化规则

```
     ┌────────────┐
     │ Controller │
     └──────┬─────┘
            │ ✅
            ▼
     ┌────────────┐     ┌──────┐
     │  Service   │────▶│  DTO │
     └──────┬─────┘     └──────┘
            │ ✅           ▲
            ▼              │ ✅
     ┌────────────┐        │
     │ Repository │────────┘
     └──────┬─────┘
            │ ✅
            ▼
     ┌────────────┐
     │   Entity   │
     └────────────┘
```

### 传递依赖（Transitive Dependencies）

用 `+` 前缀表示"传递允许"，即允许依赖该层以及该层所依赖的所有层：

```yaml
ruleset:
  Foo:
    - Bar
  Bar: ~
  Baz:
    - +Foo           # Baz 可以依赖 Foo，也可以依赖 Foo 所依赖的 Bar
  Bat:
    - Foo            # Bat 只能依赖 Foo，不能依赖 Bar
    - Bar            # 除非这里也显式声明
```

```
普通依赖 vs 传递依赖

Bat ──▶ Foo ──▶ Bar    Bat 必须声明对 Foo 和 Bar 的依赖
                       （如果 Bat 的代码用到了 Bar 的类）

Baz ──+▶ Foo ──▶ Bar   Baz 用 +Foo 即可同时访问 Foo 和 Bar
                       （自动传递）
```

## 3.6 Violation（违规）

当代码违反了 Ruleset 定义的规则时，产生 **Violation**。

### 违规的种类

```
┌──────────────────────────────────────────────────────┐
│                    分析结果类型                         │
│                                                       │
│  ┌────────────┐  代码违反了明确的规则                    │
│  │ Violation  │  例：Controller 依赖了 Repository      │
│  └────────────┘                                       │
│                                                       │
│  ┌────────────┐  代码中的某个类没有分配到任何层            │
│  │ Uncovered  │  例：App\Helper\Utils 不属于任何层      │
│  └────────────┘                                       │
│                                                       │
│  ┌────────────┐  通过 skip_violations 跳过的违规        │
│  │  Skipped   │  例：已知的遗留代码违规                   │
│  └────────────┘                                       │
│                                                       │
│  ┌────────────┐  允许的正常依赖                          │
│  │  Allowed   │  例：Controller 依赖 Service（规则允许）  │
│  └────────────┘                                       │
└──────────────────────────────────────────────────────┘
```

### 违规的输出格式

```
 src/Controller/UserController.php::12
     UserController must not depend on UserRepository (Controller on Repository)
```

各部分含义：
```
 src/Controller/UserController.php  ← 违规代码所在的文件
 ::12                               ← 行号
 UserController                     ← 产生依赖的类
 must not depend on                 ← 违规描述
 UserRepository                     ← 被依赖的类
 (Controller on Repository)         ← 涉及的层
```

### 处理违规的策略

```
                    发现违规
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │  修复代码 │ │ 调整规则 │ │ 加入跳过 │
    │          │ │          │ │          │
    │ 重构代码 │ │ 可能规则 │ │ 先记录  │
    │ 消除违规 │ │ 定义不合理│ │ 以后修复 │
    └──────────┘ └──────────┘ └──────────┘
        最优           次选        临时方案
```

## 3.7 Skip Violations（跳过违规）

对于遗留项目，不可能一次性修复所有违规。`skip_violations` 提供了渐进式修复的能力：

```yaml
deptrac:
  skip_violations:
    App\Controller\LegacyController:
      - App\Repository\UserRepository      # 跳过这个已知的违规
      - App\Repository\OrderRepository
```

### 用 Baseline 自动生成跳过列表

```bash
# 生成 baseline 文件（把所有现有违规都记录下来）
vendor/bin/deptrac --formatter=baseline --output=deptrac.baseline.yaml
```

然后在主配置中导入：

```yaml
imports:
  - deptrac.baseline.yaml

deptrac:
  paths:
    - ./src
  # ... 其余配置
```

> 💡 **工作流**：生成 baseline → 导入 baseline → 后续只检查新违规 → 逐步消除 baseline 中的违规

## 3.8 @internal 和 @deptrac-internal 注解

你可以用 PHPDoc 注解标记某些类为"层内部专用"：

```php
/**
 * @internal
 */
class InternalHelper
{
    // 这个类只应被同层的其他类使用
}
```

如果其他层的代码依赖了标记为 `@internal` 的类，Deptrac 会报告为违规。

配置中使用的标签名：

```yaml
analyser:
  internal_tag: '@internal'    # 默认值，也可以改为自定义标签如 @deptrac-internal
```

## 3.9 Uncovered Dependencies（未覆盖依赖）

当你的代码依赖了一个不属于任何层的类时，这就是"未覆盖依赖"。

```
Controller 层:  UserController ──依赖──▶ DateFormatter (← 没有属于任何层！)
```

处理方式：
1. 把 `DateFormatter` 加入一个现有层
2. 为它创建一个新层
3. 忽略它（PHP 内置类默认被忽略）

```yaml
# 控制是否忽略 PHP 内置类
deptrac:
  ignore_uncovered_internal_classes: true  # 默认为 true
```

```bash
# 运行时报告未覆盖依赖
vendor/bin/deptrac --report-uncovered

# 如果存在未覆盖依赖则返回失败退出码
vendor/bin/deptrac --fail-on-uncovered
```

## 3.10 概念关系总结图

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  PHP 源文件 ──parse──▶ Token (类/函数/文件)                   │
│                            │                                 │
│                       Collector (收集器)                      │
│                            │                                 │
│                            ▼                                 │
│                        Layer (层)                             │
│                      /     │     \                            │
│                     /      │      \                           │
│              Layer A   Layer B   Layer C                      │
│                     \      │      /                           │
│                      \     │     /                            │
│                    Ruleset (规则集)                            │
│                     "谁可以依赖谁"                             │
│                            │                                 │
│                       Analyse (分析)                          │
│                     /      │      \                           │
│                    /       │       \                          │
│            Allowed    Violation   Uncovered                   │
│            (允许)      (违规)     (未覆盖)                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 3.11 本章小结

| 概念 | 一句话解释 | 类比 |
|------|----------|------|
| **Token** | 被分析的代码单元 | 快递包裹 |
| **Collector** | 把 Token 分配到 Layer 的规则 | 快递分拣员 |
| **Layer** | 一组 Token 的逻辑分组 | 快递分拣仓库的区域 |
| **Ruleset** | 定义层间允许的依赖 | 仓库间的运输路线 |
| **Violation** | 违反规则的依赖 | 走错路线的快递 |
| **Uncovered** | 没有归属层的 Token | 没贴标签的包裹 |
| **Baseline** | 已知违规的记录 | "暂时放过"的清单 |

---

**下一章**：[第 4 章：配置文件全攻略](./04-configuration.md) — YAML 和 PHP 两种配置方式的完整指南！
