# 第 4 章：配置文件全攻略

> 🟡 适合阶段：熟悉
> 
> 阅读时间：约 15 分钟

---

## 4.1 配置文件格式选择

Deptrac 支持两种配置格式：

| 格式 | 文件名 | 优点 | 缺点 | 推荐场景 |
|------|--------|------|------|---------|
| **YAML** | `deptrac.yaml` | 简单、声明式、易读 | 不支持动态逻辑 | 大多数项目 |
| **PHP** | `deptrac.php` | 可编程、动态生成、IDE 补全 | 稍复杂 | 大型项目、需要动态配置 |

## 4.2 YAML 配置完整参考

```yaml
# ──────────────────────────────────────────
# deptrac.yaml - 完整配置参考
# ──────────────────────────────────────────

# 导入其他配置文件（在 deptrac: 之外）
imports:
  - deptrac.baseline.yaml              # 导入 baseline
  - layers/domain.yaml                  # 导入分层配置

# 所有 Deptrac 配置都在 deptrac: 下
deptrac:

  # ─── 扫描路径 ───
  paths:
    - ./src                             # 分析 src 目录
    - ./lib                             # 也可以分析多个目录

  # ─── 排除文件 ───
  exclude_files:
    - '#.*test.*#i'                     # 排除测试文件（正则）
    - '#.*fixture.*#i'                  # 排除 fixture 文件

  # ─── 架构层定义 ───
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

  # ─── 依赖规则 ───
  ruleset:
    Controller:
      - Service
    Service:
      - Repository
    Repository: ~

  # ─── 跳过的违规 ───
  skip_violations:
    App\Controller\LegacyController:
      - App\Repository\UserRepository

  # ─── 分析器配置 ───
  analyser:
    internal_tag: '@internal'           # 内部类标记
    types:                              # 要分析的依赖类型
      - class                           # 类级别依赖
      - function                        # 函数依赖
      - file                            # 文件依赖
      - function_call                   # 函数调用
      - superglobal                     # 超全局变量

  # ─── 忽略 PHP 内置类 ───
  ignore_uncovered_internal_classes: true

  # ─── 格式化器配置 ───
  formatters:
    graphviz:
      hidden_layers:
        - Vendor                        # 不在图中显示的层
      groups:
        App:                            # 分组显示
          - Controller
          - Service
          - Repository
      pointToGroups: true               # 连线指向组而非节点

    mermaidjs:
      direction: TD                     # 图的方向：TD(上→下), LR(左→右)
      groups:
        App:
          - Controller
          - Service
      hidden_layers: []

    codeclimate:
      severity:
        failure: major                  # 违规的严重级别
        skipped: minor
        uncovered: info

# ─── 自定义参数 ───
parameters:
  app_path: ./src
  domain_path: '%app_path%/Domain'

# ─── 自定义服务（高级） ───
services:
  App\Deptrac\CustomCollector:
    tags:
      - { name: 'collector', type: 'custom' }
```

## 4.3 配置文件结构图

```
deptrac.yaml
├── imports:                    ← 导入其他文件（可选，顶层）
│   └── [文件路径列表]
│
├── deptrac:                    ← 主配置（必须）
│   ├── paths:                  ← 扫描路径（必须）
│   ├── exclude_files:          ← 排除规则（可选）
│   ├── layers:                 ← 层定义（必须）
│   │   └── [层列表]
│   │       ├── name:           ← 层名称
│   │       ├── collectors:     ← 收集器列表
│   │       └── private:        ← 是否私有（可选）
│   ├── ruleset:                ← 依赖规则（必须）
│   ├── skip_violations:        ← 跳过列表（可选）
│   ├── analyser:               ← 分析器配置（可选）
│   ├── ignore_uncovered_internal_classes:  ← （可选）
│   └── formatters:             ← 格式化器配置（可选）
│
├── parameters:                 ← 自定义参数（可选，顶层）
│
└── services:                   ← 自定义服务（可选，顶层）
```

## 4.4 PHP 配置方式

PHP 配置的最大优势是 **可编程性** —— 你可以用循环、条件等 PHP 语法动态生成配置。

### 基本模板

```php
<?php
// deptrac.php

use Deptrac\Deptrac\Contract\Config\DeptracConfig;
use Deptrac\Deptrac\Contract\Config\EmitterType;
use Deptrac\Deptrac\Contract\Config\Layer;
use Deptrac\Deptrac\Contract\Config\Ruleset;
use Deptrac\Deptrac\Contract\Config\Collector\DirectoryConfig;
use Deptrac\Deptrac\Contract\Config\Collector\ClassLikeConfig;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (DeptracConfig $config, ContainerConfigurator $containerConfigurator): void {
    $config
        ->paths('src')
        ->analyser(
            AnalyserConfig::create()->types(
                EmitterType::CLASS_TOKEN,
                EmitterType::FUNCTION_TOKEN,
            )
        );

    // 定义层
    $controller = Layer::withName('Controller')->collectors(
        ClassLikeConfig::create('.*Controller.*')
    );
    $service = Layer::withName('Service')->collectors(
        ClassLikeConfig::create('.*Service.*')
    );
    $repository = Layer::withName('Repository')->collectors(
        ClassLikeConfig::create('.*Repository.*')
    );

    $config
        ->layers($controller, $service, $repository)
        ->rulesets(
            Ruleset::forLayer($controller)->accesses($service),
            Ruleset::forLayer($service)->accesses($repository),
            Ruleset::forLayer($repository),
        );
};
```

### PHP 配置的杀手级特性：动态生成

```php
<?php
// 假设你有 20 个模块，每个模块都有类似的层结构
$modules = ['User', 'Order', 'Payment', 'Product', 'Shipping'];

$layers = [];
$rulesets = [];

foreach ($modules as $module) {
    // 每个模块自动生成 3 个层
    $controller = Layer::withName("{$module}Controller")->collectors(
        DirectoryConfig::create("src/{$module}/Controller/.*")
    );
    $service = Layer::withName("{$module}Service")->collectors(
        DirectoryConfig::create("src/{$module}/Service/.*")
    );
    $repository = Layer::withName("{$module}Repository")->collectors(
        DirectoryConfig::create("src/{$module}/Repository/.*")
    );

    $layers[] = $controller;
    $layers[] = $service;
    $layers[] = $repository;

    // 每个模块内部的规则
    $rulesets[] = Ruleset::forLayer($controller)->accesses($service);
    $rulesets[] = Ruleset::forLayer($service)->accesses($repository);
    $rulesets[] = Ruleset::forLayer($repository);
}

$config
    ->layers(...$layers)
    ->rulesets(...$rulesets);
```

> 💡 **如果用 YAML 写上面的配置，需要 100+ 行。用 PHP 只需要 20 行。**

### 运行 PHP 配置

```bash
vendor/bin/deptrac -c deptrac.php
```

## 4.5 配置拆分与导入

大型项目的配置可能非常长。Deptrac 支持将配置拆分为多个文件：

```
项目根目录/
├── deptrac.yaml                    # 主配置
├── deptrac.baseline.yaml           # baseline（自动生成）
├── config/
│   ├── layers-domain.yaml          # Domain 层定义
│   └── layers-infra.yaml           # Infrastructure 层定义
```

### 主配置文件

```yaml
# deptrac.yaml
imports:
  - deptrac.baseline.yaml
  - config/layers-domain.yaml
  - config/layers-infra.yaml

deptrac:
  paths:
    - ./src

  layers:
    - name: Controller
      collectors:
        - type: classLike
          value: .*Controller.*

  ruleset:
    Controller:
      - DomainService
    DomainService:
      - DomainRepository
    InfraService:
      - DomainRepository
```

### 子配置文件

```yaml
# config/layers-domain.yaml
deptrac:
  layers:
    - name: DomainService
      collectors:
        - type: directory
          value: src/Domain/Service/.*
    - name: DomainRepository
      collectors:
        - type: directory
          value: src/Domain/Repository/.*
```

> ⚠️ **注意**：`imports` 必须放在文件**最顶层**，不在 `deptrac:` 下面！

### 导入合并规则

```
主配置的 layers + 导入文件的 layers = 最终的 layers（合并）
主配置的 ruleset + 导入文件的 ruleset = 最终的 ruleset（合并）
主配置的 skip_violations + 导入文件的 skip_violations = 最终的 skip_violations（合并）
```

## 4.6 参数系统

Deptrac 提供内置参数和自定义参数：

### 内置参数

| 参数 | 说明 |
|------|------|
| `%currentWorkingDirectory%` | 执行命令时的工作目录 |
| `%projectDirectory%` | 配置文件所在的目录 |
| `%cache_file%` | 缓存文件路径 |

### 自定义参数

```yaml
parameters:
  app_path: ./src
  domain_path: '%app_path%/Domain'

deptrac:
  paths:
    - '%app_path%'
  layers:
    - name: Domain
      collectors:
        - type: directory
          value: '%domain_path%/.*'
```

## 4.7 缓存配置

Deptrac 默认启用缓存以加速后续分析：

```bash
# 缓存行为控制
vendor/bin/deptrac                     # 使用缓存
vendor/bin/deptrac --no-cache          # 禁用缓存
vendor/bin/deptrac --clear-cache       # 清除后重建缓存
vendor/bin/deptrac --cache-file=.cache # 指定缓存文件路径
```

```yaml
# 在配置文件中指定缓存路径
parameters:
  cache_file: .deptrac.cache
```

> 💡 **建议**：把缓存文件加入 `.gitignore`：
> ```
> # .gitignore
> .deptrac.cache
> ```

## 4.8 Feature Flags（实验性特性）

```yaml
# YAML
deptrac:
  feature_flags:
    phpstan_parser: true     # 启用 PHPStan 解析器（实验性）
```

```php
// PHP
use Deptrac\Deptrac\Contract\Config\FeatureFlagsConfig;

$config->featureFlags(
    FeatureFlagsConfig::create(phpstanParser: true)
);
```

PHPStan 解析器可以提供更精确的类型推断，但可能与某些代码模式不兼容。

## 4.9 熟悉阶段常见坑

### 坑 1：imports 放错位置

```yaml
# ❌ 错误：imports 放在了 deptrac 下面
deptrac:
  imports:
    - baseline.yaml
  paths:
    - ./src

# ✅ 正确：imports 在顶层
imports:
  - baseline.yaml

deptrac:
  paths:
    - ./src
```

### 坑 2：YAML 语法错误

```yaml
# ❌ 错误：ruleset 的值不正确
ruleset:
  Controller: Service        # 错！应该是列表

# ✅ 正确
ruleset:
  Controller:
    - Service
```

### 坑 3：PHP 配置找不到类

```php
// ❌ 错误：忘记 use 语句
return static function (DeptracConfig $config): void {
    Layer::withName('Foo');  // 报错！找不到 Layer 类
};

// ✅ 正确：添加 use 语句
use Deptrac\Deptrac\Contract\Config\Layer;
```

### 坑 4：排除文件的正则写法

```yaml
exclude_files:
  # ❌ 错误：不是正则格式
  - tests/
  - vendor/

  # ✅ 正确：使用正则（带分隔符）
  - '#.*test.*#i'
  - '#.*vendor.*#i'
```

### 坑 5：多个配置文件导入时参数冲突

当多个文件定义了相同的 `paths` 时，它们会被合并。但如果你在 baseline 文件中也定义了 `paths`，可能导致意外的扫描范围。

**最佳实践**：只在主配置文件中定义 `paths`。

## 4.10 YAML vs PHP 配置选择决策树

```
开始
  │
  ├── 项目层数 < 10？
  │     ├── 是 → 用 YAML ✅
  │     └── 否 ↓
  │
  ├── 层有重复模式？（如每个模块结构类似）
  │     ├── 是 → 用 PHP ✅（动态生成）
  │     └── 否 ↓
  │
  ├── 需要条件逻辑？（如根据环境变量调整）
  │     ├── 是 → 用 PHP ✅
  │     └── 否 ↓
  │
  ├── 团队是否熟悉 PHP 配置写法？
  │     ├── 否 → 用 YAML ✅
  │     └── 是 → 用 PHP ✅（更灵活）
  │
  └── 默认推荐：YAML ✅
```

## 4.11 本章小结

| 学到了 | 要点 |
|--------|------|
| 两种配置格式 | YAML（简单声明式）和 PHP（动态可编程） |
| 完整配置结构 | paths / layers / ruleset / analyser / formatters |
| 配置拆分 | `imports` 在顶层，支持拆分到多个文件 |
| 参数系统 | 内置参数 + 自定义参数，用 `%param%` 引用 |
| 缓存管理 | `--no-cache` / `--clear-cache` / `--cache-file` |
| PHP 配置优势 | 动态生成，减少重复，适合大型项目 |

---

**下一章**：[第 5 章：收集器百科全书](./05-collectors.md) — 20+ 种收集器的详细用法和场景！
