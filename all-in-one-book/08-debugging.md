# 第 8 章：调试与排错指南

> 🟡 熟悉 → 🔴 精通
> 
> 阅读时间：约 15 分钟

---

## 8.1 调试命令全景

Deptrac 提供了一组强大的调试命令，帮你理解"为什么会这样"：

```
┌───────────────────────────────────────────────────────┐
│                   调试命令速查                          │
│                                                        │
│  "这个层匹配了哪些类？"                                  │
│  └── debug:layer                                       │
│                                                        │
│  "这个类属于哪些层？"                                    │
│  └── debug:token                                       │
│                                                        │
│  "哪些类没有被分到任何层？"                                │
│  └── debug:unassigned                                  │
│                                                        │
│  "层与层之间有哪些依赖？"                                  │
│  └── debug:dependencies                                │
│                                                        │
│  "哪些规则几乎没被用到？"                                  │
│  └── debug:unused                                      │
│                                                        │
│  "这个文件变了会影响哪些层？"                               │
│  └── changed-files (实验性)                              │
└───────────────────────────────────────────────────────┘
```

## 8.2 debug:layer — 查看层中的所有 Token

**场景**：你定义了一个层，但不确定正则是否正确匹配了想要的类。

```bash
vendor/bin/deptrac debug:layer Controller
```

**输出示例**：

```
 ───────────────────────────────────────── ───────────── 
  Token                                    Type         
 ───────────────────────────────────────── ───────────── 
  App\Controller\UserController            class-like   
  App\Controller\ProductController         class-like   
  App\Controller\OrderController           class-like   
  App\Controller\Api\V1\UserApiController  class-like   
 ───────────────────────────────────────── ───────────── 
```

**如果输出为空** → 你的收集器正则没有匹配到任何类！

### 排查正则匹配问题的流程

```
debug:layer 输出为空
        │
        ├── 检查 paths 配置是否包含了目标目录
        │
        ├── 检查 exclude_files 是否排除了目标文件
        │
        ├── 检查收集器的 value 正则是否正确
        │     ├── classLike 不需要分隔符
        │     ├── 注意 FQCN 包含命名空间
        │     └── 正则中 \ 在 YAML 中需要双写 \\
        │
        └── 检查文件中的类是否有正确的 namespace
```

## 8.3 debug:token — 查看 Token 属于哪些层

**场景**：一个类出现了意外的违规，你想知道它被分到了哪个层。

```bash
vendor/bin/deptrac debug:token 'App\Controller\UserController' class-like
```

**输出示例**：

```
 ────────────── 
  Layer        
 ────────────── 
  Controller   
  HttpLayer    
 ────────────── 
```

如果一个 Token 出现在多个层中，这可能会导致意外的规则行为。

> 💡 **常见问题**：一个类同时属于两个层，你在 ruleset 中只对其中一个层设置了规则。

## 8.4 debug:unassigned — 查看未分配的 Token

**场景**：想知道有哪些类"游离"在所有层之外。

```bash
vendor/bin/deptrac debug:unassigned
```

**输出示例**：

```
 ────────────────────────────── ───────────── 
  Token                         Type         
 ────────────────────────────── ───────────── 
  App\Helper\DateFormatter       class-like   
  App\Util\StringHelper          class-like   
  App\Config\AppConfig           class-like   
 ────────────────────────────── ───────────── 
```

> 💡 **CI 中使用**：此命令有内容输出时退出码为 `2`，可用于 CI 检查。

### 处理未分配 Token 的策略

```
发现未分配的 Token
        │
        ├── 确实应该属于某个层 → 调整收集器正则
        │
        ├── 是公共工具类 → 创建 "Common" 或 "Shared" 层
        │
        ├── 是 PHP 内置类 → 默认被忽略（ignore_uncovered_internal_classes）
        │
        └── 暂时不处理 → 不使用 --fail-on-uncovered
```

## 8.5 debug:dependencies — 查看层间依赖

**场景**：想了解某个层具体依赖了哪些其他层的哪些类。

```bash
# 查看 Controller 层的所有依赖
vendor/bin/deptrac debug:dependencies Controller

# 查看 Controller 层对 Repository 层的依赖（更精确）
vendor/bin/deptrac debug:dependencies Controller Repository
```

**输出示例**：

```
 ─────────────────────────────────────────────── ───────────────────────────────── ─────── 
  Dependent                                      on                                Line   
 ─────────────────────────────────────────────── ───────────────────────────────── ─────── 
  App\Controller\UserController                  App\Service\UserService           15     
  App\Controller\UserController                  App\Service\AuthService           16     
  App\Controller\OrderController                 App\Service\OrderService          12     
  App\Controller\OrderController                 App\Repository\OrderRepository    18     ← 违规！
 ─────────────────────────────────────────────── ───────────────────────────────── ─────── 
```

## 8.6 debug:unused — 查看未使用的规则

**场景**：配置文件中定义了很多规则，但有些规则其实从未被触发过。

```bash
vendor/bin/deptrac debug:unused

# 设置使用次数阈值
vendor/bin/deptrac debug:unused --limit=5
```

**输出示例**：

```
 ──────────── ──────────── ─────── 
  Layer        can access   count  
 ──────────── ──────────── ─────── 
  Controller   Cache        0      ← 这条规则没有被使用
  Service      Logger       2      ← 只被使用了 2 次
 ──────────── ──────────── ─────── 
```

> 💡 **用途**：清理不再需要的规则，保持配置文件精简。

## 8.7 changed-files — 变更影响分析（实验性）

**场景**：想知道修改某个文件会影响哪些架构层。

```bash
# 查看文件属于哪些层
vendor/bin/deptrac changed-files src/Service/UserService.php

# 查看连锁影响（哪些层依赖了这个文件所在的层）
vendor/bin/deptrac changed-files --with-dependencies src/Service/UserService.php
```

> ⚠️ **注意**：这是实验性功能，不受向后兼容策略保护。

## 8.8 常见问题排错手册

### 问题 1："0 violations, 0 allowed" — 什么都没分析

```
 ──── ──── ──── ──── ──── ──── ────
  0    0    0    0    0    0    0
 ──── ──── ──── ──── ──── ──── ────
```

**原因 & 排查**：

```
1. paths 配置错误，没有扫描到任何文件
   → 检查 paths 是否指向正确的目录
   → 确保路径是相对于配置文件的

2. exclude_files 过于激进，排除了所有文件
   → 临时注释掉 exclude_files 看是否有变化

3. 层的收集器没有匹配到任何类
   → 逐个用 debug:layer 检查每个层

4. 所有类都没有跨层依赖
   → 用 debug:dependencies 检查
```

### 问题 2：预期应该报违规但没有报

```
排查流程：

1. 确认"违规"的类确实被分到了预期的层
   vendor/bin/deptrac debug:token 'App\Controller\Foo' class-like
   vendor/bin/deptrac debug:token 'App\Repository\Bar' class-like

2. 确认 ruleset 中确实禁止了这种依赖
   → 记住：没有列出 = 禁止。检查是否意外地允许了

3. 确认依赖类型被启用
   → 默认只分析 class 和 function 类型
   → use 语句需要单独启用

4. 检查是否在 skip_violations 或 baseline 中
   → 可能已经被跳过了
```

### 问题 3：意外的违规（不该报的报了）

```
排查流程：

1. 确认类是否被分到了错误的层
   vendor/bin/deptrac debug:token 'App\SomeClass' class-like
   → 可能正则匹配过于宽泛

2. 确认是否有隐式依赖
   → 构造函数类型提示
   → 方法参数和返回值类型提示
   → use 语句
   → instanceof 检查
   → 静态方法调用

3. 可能是传递依赖
   → 使用 -v 详细输出查看完整依赖链
```

### 问题 4：分析速度很慢

```
优化措施：

1. 启用缓存（默认启用）
   → 首次分析较慢，后续会快很多

2. 缩小扫描范围
   → paths 只包含必要的目录
   → exclude_files 排除不需要的文件

3. 减少分析的依赖类型
   → analyser.types 只保留需要的类型

4. 清除缓存重建
   → 如果缓存损坏，用 --clear-cache

5. 确认没有扫描 vendor 目录
   → paths 不要包含 vendor/
```

### 问题 5：YAML 解析错误

```
常见错误：

1. 缩进不正确
   → YAML 使用空格，不用 Tab

2. 特殊字符未转义
   → 包含 : 或 # 的值需要引号
   → 正则中的 \ 需要双写

3. 空值表示
   → 使用 ~ 或 null，不要留空

4. 列表格式
   → 使用 - 开头的列表项
   → 不要混用 [] 和 - 格式
```

## 8.9 调试技巧进阶

### 使用 -v 详细输出

```bash
vendor/bin/deptrac -v
```

详细模式会显示更多信息，包括依赖链和分析过程。

### 组合调试命令的工作流

```
第 1 步：检查层是否正确
$ vendor/bin/deptrac debug:layer Controller
$ vendor/bin/deptrac debug:layer Service
$ vendor/bin/deptrac debug:layer Repository

第 2 步：检查是否有遗漏的类
$ vendor/bin/deptrac debug:unassigned

第 3 步：检查可疑的类属于哪些层
$ vendor/bin/deptrac debug:token 'App\SomeClass' class-like

第 4 步：查看层间具体依赖
$ vendor/bin/deptrac debug:dependencies Controller Service

第 5 步：运行完整分析
$ vendor/bin/deptrac -v
```

### 当一切都对但还是报错

```
最后的排查手段：

1. 清除缓存
   vendor/bin/deptrac --clear-cache

2. 确认 PHP 版本
   php -v  # 需要 8.2+

3. 更新 Deptrac
   composer update deptrac/deptrac

4. 检查是否有多个配置文件
   → 确认 -c 指向了正确的文件

5. 最小化复现
   → 创建一个只包含问题类的最小配置
   → 逐步添加内容直到问题出现
```

## 8.10 本章小结

| 命令 | 用途 | 使用频率 |
|------|------|---------|
| `debug:layer` | 验证层匹配了哪些类 | ⭐⭐⭐⭐⭐ |
| `debug:token` | 查看类属于哪些层 | ⭐⭐⭐⭐ |
| `debug:unassigned` | 找未分层的类 | ⭐⭐⭐⭐ |
| `debug:dependencies` | 查看层间具体依赖 | ⭐⭐⭐ |
| `debug:unused` | 找不再需要的规则 | ⭐⭐ |
| `changed-files` | 变更影响分析 | ⭐ |

---

**下一章**：[第 9 章：高级特性与扩展开发](./09-advanced.md) — 自定义收集器、事件订阅、扩展 Deptrac！
