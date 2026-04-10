# 研究笔记（编者用）

> ⚠️ 此文件供续写时参考，不是给最终读者看的。

---

## 仓库基本信息

- **仓库名**: `deptrac/deptrac`（本 fork: `propress/fork-deptrac`）
- **语言**: PHP (100%)
- **许可证**: MIT
- **PHP 要求**: ^8.2
- **当前版本**: 基于 2.x 开发分支
- **主要维护者**: patrickkusebauch（通过 GitHub Sponsors 接受赞助）
- **原始作者**: Tim Glabisch, Simon Mönch, Denis Brumann

## 项目架构

### 源码目录结构

```
src/
├── Contract/          → 公开 API（受 BC 策略保护的接口）
│   ├── Analyser/
│   ├── Ast/
│   ├── Config/
│   │   ├── Collector/  → 收集器配置类
│   │   └── Formatter/  → 格式化器配置类
│   ├── Dependency/
│   ├── Layer/
│   ├── OutputFormatter/
│   └── Result/
├── Core/              → 核心业务逻辑
│   ├── Analyser/      → DependencyLayersAnalyser 等
│   ├── Ast/           → AST 加载和解析
│   ├── Dependency/    → 依赖解析
│   ├── InputCollector/ → 文件输入收集
│   └── Layer/         → 层解析和管理
├── DefaultBehavior/   → 默认实现
│   ├── Analyser/
│   ├── Ast/
│   │   ├── Extractors/ → 各种引用提取器
│   │   └── Parser/     → NikicPhpParser, PhpStanParser
│   ├── Dependency/     → 依赖发射器
│   ├── Layer/          → 内置收集器实现
│   └── OutputFormatter/ → 内置格式化器实现
└── Supportive/        → 基础设施支持
    ├── Console/       → CLI 应用和命令
    │   ├── Command/   → AnalyseCommand, InitCommand, Debug*Command
    │   ├── Subscriber/
    │   └── Symfony/
    ├── DependencyInjection/ → 服务容器构建
    ├── File/          → 文件操作
    ├── OutputFormatter/ → 格式化器提供者
    └── Time/          → 性能计时
```

### 关键文件

| 文件 | 作用 |
|------|------|
| `deptrac` (根目录) | CLI 入口点 |
| `config/services.php` | 服务定义（100+ 个服务） |
| `config/cache.php` | 缓存相关服务 |
| `config/deptrac_template.yaml` | `init` 命令生成的模板 |
| `deptrac.php` | 项目自身的 Deptrac 配置 |
| `deptrac.baseline.yaml` | 项目自身的 baseline |
| `src/Supportive/Console/Application.php` | Symfony Console 应用类 |
| `src/Supportive/DependencyInjection/ServiceContainerBuilder.php` | 容器构建器 |

### 依赖图

```
External Dependencies:
├── composer/xdebug-handler (^3.0)
├── nikic/php-parser (^5)
├── phpdocumentor/graphviz (^2.1)
├── phpdocumentor/type-resolver (^1.9.0 || ^2.0.0)
├── phpstan/phpdoc-parser (^1.5.0 || ^2.1.0)
├── phpstan/phpstan (^2.0)
├── psr/container (^2.0)
├── psr/event-dispatcher (^1.0)
├── symfony/config (^6.4 || ^7.4 || ^8.0)
├── symfony/console (^6.4 || ^7.4 || ^8.0)
├── symfony/dependency-injection (^6.4 || ^7.4 || ^8.0)
├── symfony/event-dispatcher (^6.4 || ^7.4 || ^8.0)
├── symfony/filesystem (^6.4 || ^7.4 || ^8.0)
├── symfony/finder (^6.4 || ^7.4 || ^8.0)
└── symfony/yaml (^6.4 || ^7.4 || ^8.0)
```

## 内置收集器清单

从 `src/DefaultBehavior/Layer/` 目录提取：

1. AttributeCollector
2. BoolCollector
3. ClassCollector
4. ClassLikeCollector
5. ClassNameRegexCollector
6. ComposerCollector
7. DirectoryCollector
8. ExtendsCollector
9. FunctionNameCollector
10. GlobCollector
11. ImplementsCollector
12. InheritsCollector
13. InterfaceCollector
14. LayerCollector
15. MethodCollector
16. PhpInternalCollector
17. SuperglobalCollector
18. TagValueRegexCollector
19. TraitCollector
20. UsesCollector

## 内置格式化器清单

从 `src/DefaultBehavior/OutputFormatter/` 目录提取：

1. BaselineOutputFormatter
2. CodeclimateOutputFormatter
3. ConsoleOutputFormatter
4. GithubActionsOutputFormatter
5. GraphVizOutputDotFormatter
6. GraphVizOutputDisplayFormatter
7. GraphVizOutputHtmlFormatter
8. GraphVizOutputImageFormatter
9. JsonOutputFormatter
10. JUnitOutputFormatter
11. MermaidJsOutputFormatter
12. TableOutputFormatter

## CLI 命令清单

1. `analyse` (alias: `analyze`) — 主分析命令
2. `init` — 生成配置模板
3. `debug:layer` — 查看层中的 Token
4. `debug:token` — 查看 Token 属于的层
5. `debug:unassigned` — 查看未分配 Token
6. `debug:dependencies` — 查看层间依赖
7. `debug:unused` — 查看未使用的规则
8. `changed-files` — 变更影响分析（实验性）

## 官方文档文件清单

| 文件 | 内容 |
|------|------|
| docs/index.md | 主文档入口、安装指南、快速开始 |
| docs/concepts.md | 核心概念（Layer、Ruleset、Violation） |
| docs/configuration.md | 配置参考 |
| docs/collectors.md | 收集器参考 |
| docs/formatters.md | 格式化器参考 |
| docs/debugging.md | 调试命令 |
| docs/extending_deptrac.md | 扩展开发指南 |
| docs/upgrade.md | 升级指南 |
| docs/bc_policy.md | 向后兼容策略 |
| docs/blog.md | 博客文章索引 |
| docs/blog/2023-05-11_PHP_configuration.md | PHP 配置博客 |
| docs/blog/2025-06-28_Installing_via_bin_plugin.md | 隔离安装博客 |

## 示例配置文件清单

| 文件 | 内容 |
|------|------|
| docs/examples/ControllerServiceRepository1.depfile.yaml | 经典 MVC |
| docs/examples/ModelController1.depfile.yaml | Model-Controller |
| docs/examples/ModelController2.depfile.yaml | Model-Controller (进阶) |
| docs/examples/Fixture.depfile.yaml | 基础示例 |
| docs/examples/Parameters.depfile.yaml | 参数使用示例 |
| docs/examples/SkipViolations.yaml | 跳过违规示例 |
| docs/examples/Transitive.depfile.yaml | 传递依赖示例 |
| docs/examples/Uncovered.depfile.yaml | 未覆盖依赖示例 |
| docs/examples/implements.depfile.yaml | implements 收集器示例 |
| docs/examples/DirectoryLayer.depfile.yaml | directory 收集器示例 |
| docs/examples/GlobCollector.depfile.yaml | glob 收集器示例 |
| docs/examples/MethodNames1.depfile.yaml | method 收集器示例 |
| docs/examples/import.deptrac.yaml | 配置导入示例 |
| docs/examples/import_layer1.deptrac.yaml | 导入的层配置 1 |
| docs/examples/import_layer2.deptrac.yaml | 导入的层配置 2 |
| docs/examples/import_baseline.deptrac.yaml | 导入的 baseline |
| docs/examples/symfony_depfile.yaml | Symfony 项目示例 |
| docs/examples/sylius_depfile.yaml | Sylius 项目示例 |

## Pro 版调研结论

- 截至 2026 年 4 月，没有官方的 Pro/付费/企业版
- 所有功能都在开源版（MIT 许可）中
- 维护者通过 GitHub Sponsors 接受赞助
- 如果需要企业级功能，替代方案：SonarQube, CodeScene, Lattix 等

## 测试和构建命令

```bash
# 运行单元测试
make test
# 等价于: php -d pcov.enabled=1 -d pcov.directory=. vendor/bin/phpunit

# 运行全部 QA
make qa

# 代码风格检查
make php-cs-check

# 静态分析
make phpstan
make psalm

# 自身架构检查
make deptrac
```

## 续写建议

### 可以添加的内容

1. **更多真实项目案例** — 找几个使用 Deptrac 的知名 PHP 项目的配置
2. **性能基准测试** — 不同规模项目的分析时间
3. **与 Arkitect 的详细对比** — 代码示例级别的对比
4. **PHPStan 解析器深入** — 实际效果对比
5. **自定义扩展教程** — 更完整的端到端教程
6. **视频教程脚本** — 配合视频讲解
7. **常见架构模式库** — 更多开箱即用的配置模板
8. **Troubleshooting 数据库** — 收集用户实际遇到的问题

### 需要定期更新的内容

1. PHP 版本要求变化
2. 新增的收集器
3. 新增的格式化器
4. 配置语法变化
5. 新的 CLI 命令或选项
6. Pro 版是否发布

---

*最后更新：2026-04-10*
*研究基于 commit: b27bfc5 及之前的代码*
