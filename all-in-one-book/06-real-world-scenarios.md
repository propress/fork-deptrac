# 第 6 章：真实业务场景实战

> 🟡 熟悉 → 🔴 精通
> 
> 阅读时间：约 20 分钟

---

## 6.1 场景一：经典 MVC 架构

最常见的 PHP 项目架构，适合大多数 Web 应用。

### 项目结构

```
src/
├── Controller/
│   ├── UserController.php
│   ├── ProductController.php
│   └── OrderController.php
├── Service/
│   ├── UserService.php
│   ├── ProductService.php
│   └── OrderService.php
├── Repository/
│   ├── UserRepository.php
│   ├── ProductRepository.php
│   └── OrderRepository.php
├── Entity/
│   ├── User.php
│   ├── Product.php
│   └── Order.php
└── DTO/
    ├── UserDTO.php
    └── CreateOrderRequest.php
```

### 架构规则

```
     Controller
     │       │
     │       │ ✅
     ▼       ▼
  Service   DTO ◀── 所有层都可以用
     │       ▲
     │       │
     ▼       │
  Repository─┘
     │
     ▼
   Entity ◀── 所有层都可以用
```

### 配置文件

```yaml
# deptrac.yaml
deptrac:
  paths:
    - ./src
  exclude_files:
    - '#.*test.*#i'

  layers:
    - name: Controller
      collectors:
        - type: classLike
          value: .*Controller.*
    - name: Service
      collectors:
        - type: classLike
          value: .*Service.*
    - name: Repository
      collectors:
        - type: classLike
          value: .*Repository.*
    - name: Entity
      collectors:
        - type: directory
          value: src/Entity/.*
    - name: DTO
      collectors:
        - type: directory
          value: src/DTO/.*

  ruleset:
    Controller:
      - Service
      - DTO
      - Entity
    Service:
      - Repository
      - DTO
      - Entity
    Repository:
      - Entity
    Entity: ~
    DTO: ~
```

## 6.2 场景二：DDD（领域驱动设计）

DDD 架构的核心思想：**领域层不依赖任何外部层。**

### 项目结构

```
src/
├── Domain/                     ← 领域层（核心业务逻辑）
│   ├── User/
│   │   ├── User.php           (聚合根)
│   │   ├── UserId.php         (值对象)
│   │   ├── UserRepository.php (仓储接口)
│   │   └── UserService.php    (领域服务)
│   └── Order/
│       ├── Order.php
│       ├── OrderId.php
│       ├── OrderRepository.php
│       └── OrderService.php
│
├── Application/                ← 应用层（用例/命令处理）
│   ├── Command/
│   │   ├── CreateUserCommand.php
│   │   └── CreateUserHandler.php
│   └── Query/
│       ├── GetUserQuery.php
│       └── GetUserHandler.php
│
├── Infrastructure/             ← 基础设施层（具体实现）
│   ├── Persistence/
│   │   ├── DoctrineUserRepository.php
│   │   └── DoctrineOrderRepository.php
│   ├── Http/
│   │   ├── UserController.php
│   │   └── OrderController.php
│   └── Messaging/
│       └── RabbitMqEventBus.php
│
└── SharedKernel/               ← 共享内核（跨领域的公共代码）
    ├── EventInterface.php
    └── ValueObject.php
```

### 架构规则

```
                  ┌──────────────────┐
                  │  SharedKernel    │ ◀── 所有层都可以用
                  └──────────────────┘
                           ▲
           ┌───────────────┤
           │               │
  ┌────────┴─────────┐    │
  │     Domain       │    │     Domain 不依赖任何层！
  │  (核心业务逻辑)   │    │     （只依赖 SharedKernel）
  └────────┬─────────┘    │
           ▲               │
           │               │
  ┌────────┴─────────┐    │
  │   Application    │────┘
  │  (用例编排)       │
  └────────┬─────────┘
           ▲
           │
  ┌────────┴─────────┐
  │ Infrastructure   │
  │ (具体技术实现)    │
  └──────────────────┘
```

### 配置文件

```yaml
# deptrac.yaml - DDD 架构
deptrac:
  paths:
    - ./src

  layers:
    - name: Domain
      collectors:
        - type: directory
          value: src/Domain/.*

    - name: Application
      collectors:
        - type: directory
          value: src/Application/.*

    - name: Infrastructure
      collectors:
        - type: directory
          value: src/Infrastructure/.*

    - name: SharedKernel
      collectors:
        - type: directory
          value: src/SharedKernel/.*

  ruleset:
    # 领域层：只能依赖 SharedKernel，不依赖任何其他层！
    Domain:
      - SharedKernel

    # 应用层：可以依赖领域层和共享内核
    Application:
      - Domain
      - SharedKernel

    # 基础设施层：可以依赖所有层（它是最外层）
    Infrastructure:
      - Application
      - Domain
      - SharedKernel

    # 共享内核：不依赖任何层
    SharedKernel: ~
```

> 💡 **DDD 的黄金法则**：依赖只能从外向内。Domain 是核心，不依赖外部。
> Deptrac 帮你确保这个规则不被破坏！

## 6.3 场景三：模块化单体（Modular Monolith）

准备未来拆分微服务？先用 Deptrac 确保模块间是解耦的。

### 项目结构

```
src/
├── UserModule/
│   ├── Controller/
│   ├── Service/
│   ├── Repository/
│   └── UserModuleInterface.php    ← 模块的公共接口
│
├── OrderModule/
│   ├── Controller/
│   ├── Service/
│   ├── Repository/
│   └── OrderModuleInterface.php
│
├── PaymentModule/
│   ├── Controller/
│   ├── Service/
│   ├── Repository/
│   └── PaymentModuleInterface.php
│
└── SharedModule/
    ├── Event/
    └── Contract/
```

### 架构规则

```
   ┌────────────┐     ┌────────────┐     ┌──────────────┐
   │ UserModule │     │OrderModule │     │PaymentModule │
   └──────┬─────┘     └──────┬─────┘     └──────┬───────┘
          │                  │                   │
          │ 只通过接口交互     │                   │
          │                  │                   │
          ▼                  ▼                   ▼
   ┌─────────────────────────────────────────────────┐
   │                SharedModule                      │
   │          (事件、接口、公共合约)                     │
   └─────────────────────────────────────────────────┘
   
   UserModule ──✖──▶ OrderModule  (禁止直接依赖!)
   OrderModule ──✖──▶ UserModule  (禁止直接依赖!)
```

### 配置文件

```yaml
# deptrac.yaml - 模块化单体
deptrac:
  paths:
    - ./src

  layers:
    - name: UserModule
      collectors:
        - type: directory
          value: src/UserModule/.*

    - name: OrderModule
      collectors:
        - type: directory
          value: src/OrderModule/.*

    - name: PaymentModule
      collectors:
        - type: directory
          value: src/PaymentModule/.*

    - name: SharedModule
      collectors:
        - type: directory
          value: src/SharedModule/.*

  ruleset:
    # 每个模块只能依赖 SharedModule，不能依赖其他模块！
    UserModule:
      - SharedModule
    OrderModule:
      - SharedModule
    PaymentModule:
      - SharedModule
    SharedModule: ~
```

> 🎯 **模块化单体的目标**：每个模块理论上都可以独立拆出去成为微服务。
> Deptrac 帮你在拆分之前就验证模块是否真正独立。

## 6.4 场景四：Symfony Bundle 架构

确保 Symfony Bundle 之间保持独立。

### 项目结构

```
src/
├── Bundle/
│   ├── UserBundle/
│   │   ├── Controller/
│   │   ├── Service/
│   │   ├── Entity/
│   │   └── UserBundle.php
│   ├── CmsBundle/
│   │   ├── Controller/
│   │   ├── Service/
│   │   ├── Entity/
│   │   └── CmsBundle.php
│   └── NotificationBundle/
│       ├── Controller/
│       ├── Service/
│       └── NotificationBundle.php
└── Kernel.php
```

### 配置文件

```yaml
# deptrac.yaml - Symfony Bundle
deptrac:
  paths:
    - ./src

  layers:
    - name: UserBundle
      collectors:
        - type: classLike
          value: .*Bundle\\UserBundle\\.*
    - name: CmsBundle
      collectors:
        - type: classLike
          value: .*Bundle\\CmsBundle\\.*
    - name: NotificationBundle
      collectors:
        - type: classLike
          value: .*Bundle\\NotificationBundle\\.*

  ruleset:
    UserBundle: ~
    CmsBundle:
      - UserBundle               # CMS 允许依赖 User
    NotificationBundle:
      - UserBundle               # 通知允许依赖 User（发通知需要用户信息）
```

## 6.5 场景五：六边形架构（Hexagonal / Ports & Adapters）

### 项目结构

```
src/
├── Domain/                     ← 核心领域
│   ├── Model/
│   ├── Port/                   ← 端口（接口定义）
│   │   ├── In/                 ← 入站端口
│   │   │   └── CreateUserUseCase.php
│   │   └── Out/                ← 出站端口
│   │       └── UserStoragePort.php
│   └── Service/
│       └── UserDomainService.php
│
└── Adapter/                    ← 适配器（端口的实现）
    ├── In/                     ← 入站适配器
    │   ├── Web/
    │   │   └── UserWebController.php
    │   └── Api/
    │       └── UserApiController.php
    └── Out/                    ← 出站适配器
        ├── Persistence/
        │   └── MySqlUserRepository.php
        └── Email/
            └── SmtpNotifier.php
```

### 架构规则

```
   ┌─────────────────────────────────────────┐
   │            Adapter (适配器)               │
   │  ┌──────────┐        ┌──────────────┐   │
   │  │  In 适配  │        │  Out 适配     │   │
   │  │ (Web/API) │        │ (DB/Email)   │   │
   │  └─────┬────┘        └──────┬───────┘   │
   │        │                    │            │
   └────────┼────────────────────┼────────────┘
            │                    │
            ▼                    ▼
   ┌─────────────────────────────────────────┐
   │            Domain (核心领域)              │
   │  ┌──────────┐        ┌──────────────┐   │
   │  │  Port/In  │        │  Port/Out    │   │
   │  │ (用例接口) │        │ (存储接口)   │   │
   │  └──────────┘        └──────────────┘   │
   │                                          │
   │        ┌──────────────────┐              │
   │        │   Model/Service  │              │
   │        │   (业务逻辑)      │              │
   │        └──────────────────┘              │
   └─────────────────────────────────────────┘
   
   依赖方向：Adapter → Domain（只能从外向内！）
```

### 配置文件

```yaml
# deptrac.yaml - 六边形架构
deptrac:
  paths:
    - ./src

  layers:
    - name: DomainModel
      collectors:
        - type: directory
          value: src/Domain/Model/.*

    - name: DomainPort
      collectors:
        - type: directory
          value: src/Domain/Port/.*

    - name: DomainService
      collectors:
        - type: directory
          value: src/Domain/Service/.*

    - name: AdapterIn
      collectors:
        - type: directory
          value: src/Adapter/In/.*

    - name: AdapterOut
      collectors:
        - type: directory
          value: src/Adapter/Out/.*

  ruleset:
    DomainModel: ~                      # Model 不依赖任何层
    DomainPort:
      - DomainModel                     # Port 可以引用 Model
    DomainService:
      - DomainModel                     # Service 依赖 Model
      - DomainPort                      # Service 依赖 Port
    AdapterIn:
      - DomainPort                      # 入站适配器依赖入站端口
      - DomainModel                     # 可能需要返回 Model
    AdapterOut:
      - DomainPort                      # 出站适配器实现出站端口
      - DomainModel                     # 需要操作 Model
```

## 6.6 场景六：Laravel 应用

### 项目结构

```
app/
├── Http/
│   ├── Controllers/
│   ├── Middleware/
│   └── Requests/
├── Services/
├── Repositories/
├── Models/
├── Events/
├── Listeners/
└── Providers/
```

### 配置文件

```yaml
# deptrac.yaml - Laravel
deptrac:
  paths:
    - ./app

  layers:
    - name: Controller
      collectors:
        - type: directory
          value: app/Http/Controllers/.*

    - name: Middleware
      collectors:
        - type: directory
          value: app/Http/Middleware/.*

    - name: Request
      collectors:
        - type: directory
          value: app/Http/Requests/.*

    - name: Service
      collectors:
        - type: directory
          value: app/Services/.*

    - name: Repository
      collectors:
        - type: directory
          value: app/Repositories/.*

    - name: Model
      collectors:
        - type: directory
          value: app/Models/.*

    - name: Event
      collectors:
        - type: directory
          value: app/Events/.*

    - name: Listener
      collectors:
        - type: directory
          value: app/Listeners/.*

  ruleset:
    Controller:
      - Service
      - Request
      - Model
      - Event
    Middleware:
      - Service
      - Model
    Service:
      - Repository
      - Model
      - Event
    Repository:
      - Model
    Model: ~
    Request: ~
    Event: ~
    Listener:
      - Service
      - Model
      - Event
```

## 6.7 场景选择指导

```
你的项目是什么类型？
        │
        ├── 传统 MVC → 场景一
        │
        ├── DDD 架构 → 场景二
        │
        ├── 准备拆微服务 → 场景三（模块化单体）
        │
        ├── Symfony 多 Bundle → 场景四
        │
        ├── 六边形/Clean Architecture → 场景五
        │
        └── Laravel 项目 → 场景六
```

## 6.8 场景进阶：组合使用

实际项目往往是多种模式的组合。例如：**模块化单体 + 每个模块内部用 DDD**

```yaml
deptrac:
  paths:
    - ./src

  layers:
    # 模块级别
    - name: UserModule_Domain
      collectors:
        - type: directory
          value: src/User/Domain/.*

    - name: UserModule_Application
      collectors:
        - type: directory
          value: src/User/Application/.*

    - name: UserModule_Infrastructure
      collectors:
        - type: directory
          value: src/User/Infrastructure/.*

    - name: OrderModule_Domain
      collectors:
        - type: directory
          value: src/Order/Domain/.*

    - name: OrderModule_Application
      collectors:
        - type: directory
          value: src/Order/Application/.*

    - name: OrderModule_Infrastructure
      collectors:
        - type: directory
          value: src/Order/Infrastructure/.*

  ruleset:
    # User 模块内部 DDD 规则
    UserModule_Domain: ~
    UserModule_Application:
      - UserModule_Domain
    UserModule_Infrastructure:
      - UserModule_Application
      - UserModule_Domain

    # Order 模块内部 DDD 规则
    OrderModule_Domain: ~
    OrderModule_Application:
      - OrderModule_Domain
    OrderModule_Infrastructure:
      - OrderModule_Application
      - OrderModule_Domain

    # 跨模块规则：禁止！
    # User 和 Order 之间不能有直接依赖
```

> 💡 **Pro Tip**：当层数很多时（如上面有 6 个层），强烈建议使用 PHP 配置动态生成！

## 6.9 本章小结

| 场景 | 核心规则 | 推荐收集器 |
|------|---------|----------|
| MVC | Controller → Service → Repository | `classLike` |
| DDD | 依赖从外向内，Domain 不依赖外部 | `directory` |
| 模块化单体 | 模块间只通过共享模块交互 | `directory` |
| Symfony Bundle | Bundle 间保持独立 | `classLike` |
| 六边形架构 | Adapter → Domain，不可反向 | `directory` |
| Laravel | Controller → Service → Repository → Model | `directory` |

---

**下一章**：[第 7 章：CI/CD 集成指南](./07-ci-cd-integration.md) — 把 Deptrac 集成到你的自动化流水线中！
