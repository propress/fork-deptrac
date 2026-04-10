# 第 5 章：收集器百科全书

> 🟡 适合阶段：熟悉
> 
> 阅读时间：约 20 分钟

---

## 5.1 收集器总览

收集器（Collector）是决定 "哪些代码属于哪个层" 的核心组件。

```
┌───────────────────────────────────────────────────────┐
│                     收集器分类                          │
│                                                        │
│  📝 按名称匹配          📂 按位置匹配                    │
│  ├── classLike         ├── directory                   │
│  ├── class             ├── glob                        │
│  ├── interface         └── composer                    │
│  ├── trait                                             │
│  ├── classNameRegex    🔗 按关系匹配                    │
│  ├── functionName      ├── extends                     │
│  └── tagValueRegex     ├── implements                  │
│                        ├── inherits                    │
│  🏷 按注解/属性         └── uses (trait)                │
│  └── attribute                                         │
│                        🧩 组合/特殊                     │
│  🌐 按类型              ├── bool                       │
│  ├── superglobal       ├── layer                       │
│  └── php_internal      └── method                      │
└───────────────────────────────────────────────────────┘
```

## 5.2 按名称匹配的收集器

### classLike — 最常用的收集器 ⭐⭐⭐⭐⭐

匹配类、接口、Trait、Enum，基于全限定类名（FQCN）的正则表达式。

```yaml
- type: classLike
  value: .*Controller.*          # 不需要正则分隔符
```

**内部行为**：Deptrac 会将 value 处理为 `/YOUR_VALUE/i`（大小写不敏感）。

**匹配示例**：

| 你的 value | 能匹配 | 不能匹配 |
|-----------|--------|---------|
| `.*Controller.*` | `App\UserController` | `App\UserService` |
| `App\\Controller\\.*` | `App\Controller\UserCtrl` | `Lib\Controller\Foo` |
| `.*\\Entity\\.*` | `App\Entity\User` | `App\Model\User` |

### class — 仅匹配类（排除接口/Trait/Enum）

```yaml
- type: class
  value: .*Repository.*
```

> 💡 **什么时候用 class 而不是 classLike？**
> 当你有 `UserRepository`（类）和 `UserRepositoryInterface`（接口），
> 只想在层中包含具体类而不包含接口时。

### interface — 仅匹配接口

```yaml
- type: interface
  value: .*Interface$            # 匹配以 Interface 结尾的
```

### trait — 仅匹配 Trait

```yaml
- type: trait
  value: .*Trait$
```

### classNameRegex — 完全控制正则

与 `classLike` 不同，你需要提供完整的正则表达式（含分隔符）：

```yaml
- type: classNameRegex
  value: '#^App\\Controller\\[A-Z][a-z]+Controller$#'   # 严格匹配
```

> 💡 **classLike vs classNameRegex**：
> - `classLike`：简单场景，Deptrac 帮你加分隔符
> - `classNameRegex`：需要精确控制正则时使用

### functionName — 匹配函数

```yaml
- type: functionName
  value: .*helper.*             # 匹配函数名包含 helper 的
```

> ⚠️ 需要在 `analyser.types` 中启用 `function` 类型。

### tagValueRegex — 按 PHPDoc 标签匹配

```yaml
- type: tagValueRegex
  value: '@deprecated.*'        # 匹配有 @deprecated 标签的类
```

用于收集标注了特定 PHPDoc 标签的类。

## 5.3 按位置匹配的收集器

### directory — 按目录路径匹配 ⭐⭐⭐⭐

```yaml
- type: directory
  value: src/Domain/.*          # 匹配 src/Domain/ 下的所有类
```

**匹配示例**：

```
项目结构:
src/
├── Domain/
│   ├── User/
│   │   ├── UserEntity.php      ← ✅ 匹配
│   │   └── UserService.php     ← ✅ 匹配
│   └── Order/
│       └── OrderEntity.php     ← ✅ 匹配
├── Infrastructure/
│   └── UserRepo.php            ← ❌ 不匹配
```

> 💡 **classLike vs directory 如何选择？**
> 
> | 场景 | 推荐 |
> |------|------|
> | 按命名约定分层（Controller/Service/Repository） | `classLike` |
> | 按目录结构分层（Domain/Infrastructure） | `directory` |
> | 两者结合 | 用 `bool` 收集器 |

### glob — 按 glob 模式匹配文件

```yaml
- type: glob
  value: src/Legacy/**/*.php     # glob 语法
```

### composer — 按 Composer 包匹配

从 `composer.json` / `composer.lock` 中分析 PSR-0/PSR-4 的自动加载配置。

```yaml
- type: composer
  composerPath: composer.json
  composerLockPath: composer.lock
  packages:
    - symfony/console              # 匹配 symfony/console 包中的类
    - doctrine/orm
```

**典型场景**：控制对第三方库的直接依赖。

```yaml
layers:
  - name: SymfonyConsole
    collectors:
      - type: composer
        composerPath: composer.json
        composerLockPath: composer.lock
        packages:
          - symfony/console

  - name: AppCode
    collectors:
      - type: directory
        value: src/.*

ruleset:
  AppCode:
    - SymfonyConsole            # 只有 AppCode 能用 Symfony Console
```

## 5.4 按关系匹配的收集器

### implements — 按接口实现匹配

```yaml
- type: implements
  value: App\Contract\EventHandlerInterface
```

收集所有实现了指定接口的类（递归查找）。

```
               EventHandlerInterface
               /                    \
     UserEventHandler         OrderEventHandler
           |                        |
     SpecialUserHandler      ← 也会被收集到（递归！）
```

### extends — 按继承关系匹配

```yaml
- type: extends
  value: App\Base\AbstractRepository
```

收集所有继承了指定类的类（递归查找）。

### inherits — 按继承链匹配（接口+类+Trait）

```yaml
- type: inherits
  value: App\Contract\Loggable
```

比 `extends` 和 `implements` 更广：检查所有继承关系（类继承、接口实现、Trait 使用）。

### uses — 按 Trait 使用匹配

```yaml
- type: uses
  value: App\Traits\HasTimestamps
```

收集所有使用了指定 Trait 的类。

## 5.5 按注解/属性匹配的收集器

### attribute — 按 PHP 8 属性匹配

```yaml
- type: attribute
  value: Symfony\Component\Routing\Annotation\Route
```

收集使用了指定 PHP 属性（Attribute）的类/函数/文件。

**实用场景**：收集所有标注了路由的控制器：

```php
#[Route('/users')]
class UserController { /* ... */ }
```

## 5.6 组合与特殊收集器

### bool — 组合逻辑收集器 ⭐⭐⭐

最强大的收集器之一，用 `must` (AND) 和 `must_not` (NOT) 组合多个条件：

```yaml
- type: bool
  must:                               # 必须同时满足（AND）
    - type: classLike
      value: .*Service.*
    - type: directory
      value: src/Domain/.*
  must_not:                           # 必须不满足（NOT）
    - type: classLike
      value: .*Abstract.*
    - type: classLike
      value: .*Interface.*
```

```
逻辑表达式:
  (名称匹配 *Service*) AND (在 src/Domain/ 目录下)
  AND NOT (名称匹配 *Abstract*) AND NOT (名称匹配 *Interface*)
```

**经典场景**：DDD 架构中的 Domain Service

```yaml
layers:
  - name: DomainService
    collectors:
      - type: bool
        must:
          - type: directory
            value: src/Domain/.*
          - type: classLike
            value: .*Service.*
        must_not:
          - type: implements
            value: App\Infrastructure\ExternalServiceInterface
```

### layer — 引用其他层

```yaml
layers:
  - name: AllServices
    collectors:
      - type: classLike
        value: .*Service.*

  - name: PublicServices
    collectors:
      - type: bool
        must:
          - type: layer
            value: AllServices          # 引用另一个层的所有 Token
        must_not:
          - type: classLike
            value: .*Internal.*
```

### method — 按方法名匹配

匹配包含特定方法名的类：

```yaml
- type: method
  value: .*handle.*                    # 匹配有 handle 方法的类
```

**场景**：收集所有命令处理器（有 `handle` 方法的类）。

### superglobal — 匹配超全局变量

```yaml
- type: superglobal
  value:
    - _POST
    - _GET
    - _REQUEST
```

**场景**：确保只有特定层可以访问超全局变量。

```yaml
layers:
  - name: HttpInput
    collectors:
      - type: superglobal
        value:
          - _POST
          - _GET

ruleset:
  Controller:
    - HttpInput                        # 只有 Controller 可以访问
  Service: ~                           # Service 不能访问超全局变量
```

### php_internal — PHP 内置类和函数

```yaml
- type: php_internal
  value:
    - DateTime
    - json_encode
```

## 5.7 私有收集器（Private）

给层添加 `private: true`，该层内的类只能被同层其他类引用：

```yaml
layers:
  - name: DomainInternal
    collectors:
      - type: directory
        value: src/Domain/Internal/.*
    private: true
```

```
✅ src/Domain/Internal/Helper.php → src/Domain/Internal/Utils.php  (同层，允许)
❌ src/Service/UserService.php → src/Domain/Internal/Helper.php    (跨层引用私有，违规!)
```

## 5.8 收集器选择决策指南

```
需要把代码分到某个层中
            │
            ├── 按类名/命名约定分？
            │     ├── 简单匹配 → classLike ✅
            │     └── 精确正则 → classNameRegex ✅
            │
            ├── 按目录结构分？
            │     ├── 正则路径 → directory ✅
            │     └── Glob 模式 → glob ✅
            │
            ├── 按代码关系分？
            │     ├── 实现接口 → implements ✅
            │     ├── 继承类 → extends ✅
            │     ├── 使用 Trait → uses ✅
            │     └── 含指定方法 → method ✅
            │
            ├── 按第三方依赖分？
            │     └── Composer 包 → composer ✅
            │
            ├── 按注解/属性分？
            │     └── PHP 8 Attribute → attribute ✅
            │
            ├── 需要组合多个条件？
            │     └── 用 bool (must / must_not) ✅
            │
            └── 引用另一个层？
                  └── layer ✅
```

## 5.9 收集器使用最佳实践

### 1. 从简单开始

```yaml
# ✅ 好：先用最简单的收集器
- type: classLike
  value: .*Controller.*

# ❌ 避免：一开始就用过于复杂的组合
- type: bool
  must:
    - type: classNameRegex
      value: '#^App\\(Http|Api)\\Controller\\.*#'
    - type: attribute
      value: Symfony\Component\Routing\Annotation\Route
  must_not:
    - type: extends
      value: App\Base\AbstractController
```

### 2. 正则要精确但不过度

```yaml
# ⚠️ 太宽泛：可能误匹配
- type: classLike
  value: .*Service.*           # 会匹配到 ServiceProvider, ServiceException 等

# ✅ 更精确
- type: classLike
  value: .*\\Service\\.*       # 匹配 Service 命名空间下的类
```

### 3. 用 debug:layer 验证

```bash
# 配完收集器后一定要验证！
vendor/bin/deptrac debug:layer Controller
vendor/bin/deptrac debug:layer Service
```

### 4. 多个收集器是 OR 关系

```yaml
# 这个层包含：所有 Repository 类 OR 所有在 storage/ 目录下的类
- name: DataAccess
  collectors:
    - type: classLike
      value: .*Repository.*
    - type: directory
      value: src/Storage/.*
```

## 5.10 本章小结

| 收集器 | 匹配依据 | 最佳场景 |
|--------|---------|---------|
| `classLike` | 类名正则 | 通用首选 |
| `directory` | 文件路径 | 按目录组织的项目 |
| `bool` | 组合条件 | 复杂过滤 |
| `implements` | 接口实现 | 接口驱动架构 |
| `extends` | 类继承 | 基于继承的架构 |
| `composer` | Composer 包 | 第三方依赖管控 |
| `attribute` | PHP 属性 | 使用 Attribute 的项目 |
| `class`/`interface`/`trait` | 按类型过滤 | 需要区分类型时 |
| `superglobal` | PHP 超全局变量 | 限制全局变量访问 |
| `method` | 方法名 | 按行为分组 |
| `layer` | 引用其他层 | 构建层的差集 |

---

**下一章**：[第 6 章：真实业务场景实战](./06-real-world-scenarios.md) — MVC、DDD、模块化单体等场景的完整配置！
