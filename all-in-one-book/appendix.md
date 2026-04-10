# 附录：Pro 版对比、FAQ、速查表

> 🟢🟡🔴 适合阶段：全阶段
> 
> 阅读时间：按需查阅

---

## A.1 Pro 版 vs 开源版对比

### 当前状态（截至 2026 年）

经过对官方仓库、官方文档、GitHub、Packagist 等渠道的全面调研：

> **Deptrac 目前没有官方的"Pro"付费版本。** 它是一个纯开源工具（MIT 许可），
> 所有功能都包含在免费版中。

### 功能完整度分析

| 功能类别 | 开源版是否包含 | 说明 |
|---------|--------------|------|
| 核心架构分析 | ✅ 完全包含 | 层定义、规则检查、违规检测 |
| 20+ 种收集器 | ✅ 完全包含 | classLike, directory, bool, implements 等 |
| 12+ 种格式化器 | ✅ 完全包含 | table, json, junit, graphviz, mermaid 等 |
| CI/CD 集成 | ✅ 完全包含 | GitHub Actions, GitLab CI 原生支持 |
| 可视化 | ✅ 完全包含 | Graphviz + Mermaid.js |
| Baseline 机制 | ✅ 完全包含 | 渐进式修复支持 |
| 自定义扩展 | ✅ 完全包含 | 收集器、格式化器、事件订阅者 |
| YAML + PHP 配置 | ✅ 完全包含 | 两种配置方式 |
| 调试命令 | ✅ 完全包含 | debug:layer, debug:token 等 |
| 缓存优化 | ✅ 完全包含 | AST 缓存加速 |

### 什么情况下你可能需要商业级方案？

如果你需要以下功能，Deptrac 开源版 **无法满足**，需要考虑其他商业工具：

| 需求 | Deptrac 现状 | 替代方案 |
|------|-------------|---------|
| 专业技术支持（SLA） | 社区支持 | 联系维护者商议定制支持 |
| Web 可视化仪表板 | 仅命令行图表 | SonarQube, CodeScene |
| 历史趋势分析 | 无 | CodeScene, Ndepend (C#) |
| 团队协作/权限管理 | 无 | SonarQube |
| 跨语言架构分析 | 仅 PHP | Lattix, Structure101 |

### 支持项目发展

Deptrac 的主要维护者通过 GitHub Sponsors 接受赞助：

```
如果 Deptrac 对你的团队有价值，考虑通过 GitHub Sponsors 赞助维护者。
这有助于项目持续发展和获得更好的支持。
```

### 结论：是否值得付费？

```
你的需求                                推荐
────────────────────────────────────────────────────────
PHP 架构检查                        → Deptrac 开源版 ✅ (免费)
+ CI/CD 集成                       → Deptrac 开源版 ✅ (免费)
+ 可视化                           → Deptrac 开源版 ✅ (免费)
+ 团队协作仪表板                    → SonarQube Community ✅ (免费)
+ 历史趋势 + 技术债务追踪           → CodeScene / SonarQube 💰 (付费)
+ 企业级支持 SLA                   → 商业工具 💰 (付费)
```

---

## A.2 常见问题 FAQ

### 安装相关

**Q: Deptrac 支持哪些 PHP 版本？**

A: Deptrac 本身需要 PHP 8.2+ 运行。但它可以分析使用更低 PHP 版本编写的项目代码（只要 nikic/php-parser 能解析）。

**Q: 安装时 Composer 报依赖冲突怎么办？**

A: 使用 `bamarni/composer-bin-plugin` 隔离安装，或者下载 PHAR 文件。详见第 2 章。

**Q: 可以全局安装吗？**

A: 可以。通过 PHIVE (`phive install -g deptrac/deptrac`) 或手动下载 PHAR 并放到 PATH 中。

### 配置相关

**Q: 配置文件应该叫什么名字？**

A: 默认为 `deptrac.yaml`，也可以使用 `deptrac.php`。通过 `-c` 参数指定其他名称。

**Q: YAML 和 PHP 配置可以混用吗？**

A: 主配置只能选一种格式。但 YAML 主配置可以通过 `imports` 导入其他 YAML 文件。

**Q: 一个类可以属于多个层吗？**

A: 可以。如果一个类被多个层的收集器匹配到，它就属于多个层。规则会分别应用。

**Q: `~` 在 YAML 中是什么意思？**

A: `~` 是 YAML 的 `null` 值。在 ruleset 中表示该层不允许依赖任何其他层。

### 使用相关

**Q: Deptrac 会修改我的代码吗？**

A: 绝对不会。Deptrac 是只读的分析工具，它只报告问题，不改变任何代码。

**Q: 分析速度如何？**

A: 首次分析可能需要几秒到几十秒（取决于项目大小）。后续分析由于缓存，通常只需 1-2 秒。

**Q: 我应该把 `.deptrac.cache` 提交到 Git 吗？**

A: 不应该。缓存文件应加入 `.gitignore`。

**Q: Deptrac 能分析 vendor 目录吗？**

A: 技术上可以（把 vendor 加到 paths 中），但通常不建议。vendor 中的代码不是你能控制的。

**Q: 如何处理泛型/模板类型？**

A: 启用 PHPStan 解析器（`feature_flags.phpstan_parser: true`）可以更好地解析泛型类型。

### CI 相关

**Q: Deptrac 的退出码是什么？**

A: `0` = 无违规，`1` = 有违规。`debug:unassigned` 有输出时返回 `2`。

**Q: GitHub Actions 中如何在 PR 代码行上显示错误？**

A: Deptrac 会自动检测 GitHub Actions 环境并启用 `github-actions` 格式化器，无需额外配置。

---

## A.3 命令速查表

### 核心命令

```bash
# 运行分析（默认命令）
vendor/bin/deptrac

# 指定配置文件
vendor/bin/deptrac -c deptrac.php

# 详细输出
vendor/bin/deptrac -v

# 禁用缓存
vendor/bin/deptrac --no-cache

# 清除缓存
vendor/bin/deptrac --clear-cache

# 指定缓存文件
vendor/bin/deptrac --cache-file=.cache/deptrac.cache

# 生成配置模板
vendor/bin/deptrac init
```

### 输出控制

```bash
# 指定格式化器
vendor/bin/deptrac --formatter=json
vendor/bin/deptrac --formatter=junit
vendor/bin/deptrac --formatter=table
vendor/bin/deptrac --formatter=console
vendor/bin/deptrac --formatter=codeclimate
vendor/bin/deptrac --formatter=mermaidjs

# 输出到文件
vendor/bin/deptrac --formatter=json --output=report.json

# 生成 baseline
vendor/bin/deptrac --formatter=baseline --output=deptrac.baseline.yaml

# 生成架构图
vendor/bin/deptrac --formatter=graphviz-image --output=arch.png
vendor/bin/deptrac --formatter=graphviz-dot --output=arch.dot
vendor/bin/deptrac --formatter=graphviz-html --output=arch.html
vendor/bin/deptrac --formatter=graphviz-display
```

### CI/CD 选项

```bash
# 禁用进度条（CI 推荐）
vendor/bin/deptrac --no-progress

# 报告未覆盖依赖
vendor/bin/deptrac --report-uncovered

# 未覆盖依赖时失败
vendor/bin/deptrac --fail-on-uncovered

# 报告跳过的违规
vendor/bin/deptrac --report-skipped
```

### 调试命令

```bash
# 查看层中的所有 Token
vendor/bin/deptrac debug:layer LayerName

# 查看 Token 属于哪些层
vendor/bin/deptrac debug:token 'Full\ClassName' class-like

# 查看未分配的 Token
vendor/bin/deptrac debug:unassigned

# 查看层间依赖
vendor/bin/deptrac debug:dependencies LayerA
vendor/bin/deptrac debug:dependencies LayerA LayerB

# 查看未使用的规则
vendor/bin/deptrac debug:unused
vendor/bin/deptrac debug:unused --limit=5

# 查看变更影响（实验性）
vendor/bin/deptrac changed-files src/file.php
vendor/bin/deptrac changed-files --with-dependencies src/file.php
```

---

## A.4 配置速查表

### 最小配置

```yaml
deptrac:
  paths:
    - ./src
  layers:
    - name: LayerA
      collectors:
        - type: classLike
          value: .*PatternA.*
    - name: LayerB
      collectors:
        - type: classLike
          value: .*PatternB.*
  ruleset:
    LayerA:
      - LayerB
    LayerB: ~
```

### 完整配置

```yaml
imports:
  - deptrac.baseline.yaml

deptrac:
  paths:
    - ./src
  exclude_files:
    - '#.*test.*#i'
  layers:
    - name: Layer
      collectors:
        - type: classLike
          value: .*Pattern.*
      private: false
  ruleset:
    Layer:
      - OtherLayer
      - +TransitiveLayer
  skip_violations:
    Full\ClassName:
      - Other\ClassName
  analyser:
    internal_tag: '@internal'
    types:
      - class
      - function
      - function_call
      - file
      - superglobal
  ignore_uncovered_internal_classes: true
  feature_flags:
    phpstan_parser: false
  formatters:
    graphviz:
      hidden_layers: []
      groups:
        GroupName:
          - Layer1
          - Layer2
      pointToGroups: false
    mermaidjs:
      direction: TD
      groups: {}
      hidden_layers: []
    codeclimate:
      severity:
        failure: major
        skipped: minor
        uncovered: info

parameters:
  custom_param: value

services:
  App\Custom\Collector:
    tags:
      - { name: 'collector', type: 'custom' }
```

---

## A.5 收集器速查表

| 收集器 | 语法 | 匹配 |
|--------|------|------|
| `classLike` | `value: .*Regex.*` | 类/接口/Trait/Enum 名 |
| `class` | `value: .*Regex.*` | 仅类名 |
| `interface` | `value: .*Regex.*` | 仅接口名 |
| `trait` | `value: .*Regex.*` | 仅 Trait 名 |
| `classNameRegex` | `value: '#regex#'` | 类名（完整正则） |
| `functionName` | `value: .*Regex.*` | 函数名 |
| `tagValueRegex` | `value: '@tag.*'` | PHPDoc 标签 |
| `directory` | `value: src/Dir/.*` | 文件路径 |
| `glob` | `value: src/**/*.php` | Glob 模式 |
| `composer` | `packages: [pkg/name]` | Composer 包 |
| `implements` | `value: Full\Interface` | 接口实现 |
| `extends` | `value: Full\Class` | 类继承 |
| `inherits` | `value: Full\Name` | 所有继承关系 |
| `uses` | `value: Full\Trait` | Trait 使用 |
| `attribute` | `value: Full\Attr` | PHP 8 属性 |
| `bool` | `must: [...], must_not: [...]` | 组合条件 |
| `layer` | `value: LayerName` | 引用其他层 |
| `method` | `value: .*handle.*` | 方法名 |
| `superglobal` | `value: [_POST, _GET]` | PHP 超全局变量 |
| `php_internal` | `value: [DateTime]` | PHP 内置类/函数 |

---

## A.6 Deptrac 生态相关工具

| 工具 | 用途 | 搭配 Deptrac |
|------|------|-------------|
| **PHPStan** | 静态类型分析 | Deptrac 检查架构，PHPStan 检查类型 |
| **Psalm** | 静态类型分析 | 同 PHPStan |
| **PHP-CS-Fixer** | 代码风格修复 | Deptrac 管架构，CS-Fixer 管风格 |
| **PHPUnit** | 单元测试 | 测试行为正确性 |
| **Infection** | 变异测试 | 测试质量验证 |
| **Rector** | 自动重构 | 可帮助修复 Deptrac 发现的架构问题 |
| **Graphviz** | 图形可视化 | 生成架构依赖图 |

### 推荐的 PHP 项目质量工具链

```
┌──────────┐  ┌───────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐
│ PHP-CS   │  │  Deptrac  │  │ PHPStan │  │ PHPUnit │  │Infection │
│ Fixer    │  │           │  │         │  │         │  │          │
│          │  │           │  │         │  │         │  │          │
│ 代码风格 │  │ 架构规则   │  │ 类型安全 │  │ 功能测试 │  │ 测试质量 │
└──────────┘  └───────────┘  └─────────┘  └─────────┘  └──────────┘
     ▼              ▼              ▼            ▼            ▼
                    CI / CD 流水线中依次运行
```

---

*本附录将随项目发展持续更新。*
