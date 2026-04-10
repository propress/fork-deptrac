# 第 9 章：高级特性与扩展开发

> 🔴 适合阶段：精通
> 
> 阅读时间：约 15 分钟

---

## 9.1 Deptrac 内部分析流水线

在学习扩展之前，先理解 Deptrac 的内部工作流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Deptrac 分析流水线                              │
│                                                                  │
│  ① 文件发现           ② AST 解析           ③ 引用提取             │
│  ┌──────────┐       ┌──────────┐        ┌──────────────┐        │
│  │ Finder   │──────▶│ PHP      │───────▶│ Reference    │        │
│  │ (找文件)  │       │ Parser   │        │ Extractors   │        │
│  └──────────┘       │(解析AST) │        │(提取类引用)    │        │
│                     └──────────┘        └──────┬───────┘        │
│                                                │                 │
│  ④ 依赖发射           ⑤ 层分配             ⑥ 规则检查             │
│  ┌──────────────┐  ┌──────────┐        ┌──────────────┐        │
│  │ Dependency   │  │ Layer    │        │ Analyser     │        │
│  │ Emitters     │──│ Resolver │───────▶│ (规则检查)    │        │
│  │(生成依赖关系) │  │(分配到层) │        └──────┬───────┘        │
│  └──────────────┘  └──────────┘               │                 │
│                                                │                 │
│  ⑦ 结果输出                                                      │
│  ┌──────────────┐                                                │
│  │ Output       │◀────────────────────────────┘                 │
│  │ Formatters   │                                                │
│  │(格式化输出)   │                                                │
│  └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

每个步骤都有对应的扩展点。

## 9.2 扩展点总览

| 扩展点 | 接口 | 用途 | 难度 |
|--------|------|------|------|
| **自定义收集器** | `CollectorInterface` | 新的代码分组方式 | ⭐⭐ |
| **引用提取器** | `ReferenceExtractorInterface` | 提取新类型的依赖引用 | ⭐⭐⭐ |
| **依赖发射器** | `DependencyEmitterInterface` | 新的依赖关系类型 | ⭐⭐⭐ |
| **事件订阅者** | `EventSubscriberInterface` | 自定义分析规则 | ⭐⭐⭐⭐ |
| **输出格式化器** | `OutputFormatterInterface` | 新的输出格式 | ⭐⭐ |
| **AST 解析器** | `ParserInterface` | 替换默认的 PHP 解析器 | ⭐⭐⭐⭐⭐ |
| **Baseline 映射器** | `BaselineMapperInterface` | 自定义 baseline 格式 | ⭐⭐⭐ |

## 9.3 自定义收集器（最常用的扩展）

当内置的 20+ 种收集器都不能满足需求时，你可以创建自己的收集器。

### 示例：按文件大小收集

```php
<?php
namespace App\Deptrac;

use Deptrac\Deptrac\Contract\Layer\CollectorInterface;
use Deptrac\Deptrac\Contract\Layer\TokenReference;

class FileSizeCollector implements CollectorInterface
{
    public function satisfy(
        array $config,
        TokenReference $reference
    ): bool {
        // config 来自 YAML 配置中的参数
        $maxSize = $config['maxSize'] ?? 1000;
        
        $filePath = $reference->getFilePath();
        if ($filePath === null) {
            return false;
        }
        
        return filesize($filePath) > $maxSize;
    }
}
```

### 注册收集器

在 `deptrac.yaml` 的 `services` 部分注册：

```yaml
services:
  App\Deptrac\FileSizeCollector:
    tags:
      - { name: 'collector', type: 'fileSize' }
```

### 使用自定义收集器

```yaml
deptrac:
  layers:
    - name: LargeFiles
      collectors:
        - type: fileSize          # 使用你注册的 type 名
          maxSize: 5000           # 传递配置参数
```

## 9.4 自定义事件订阅者（自定义规则）

事件订阅者让你可以实现自定义的违规检查逻辑。

### Deptrac 的事件系统

```
分析每个依赖时 ───▶ ProcessEvent (逐个依赖)
分析全部完成后 ───▶ PostProcessEvent (最终处理)
```

### 示例：禁止依赖包含 "Legacy" 的类

```php
<?php
namespace App\Deptrac;

use Deptrac\Deptrac\Contract\Analyser\ProcessEvent;
use Deptrac\Deptrac\Contract\Analyser\ViolationCreatingInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class ForbidLegacyDependencyRule implements EventSubscriberInterface, ViolationCreatingInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ProcessEvent::class => 'onProcess',
        ];
    }

    public function onProcess(ProcessEvent $event): void
    {
        $dependency = $event->dependency;
        $dependentName = $dependency->getDependant()->toString();
        $dependeeeName = $dependency->getDependee()->toString();

        // 如果依赖了包含 "Legacy" 的类，报告违规
        if (str_contains($dependeeeName, 'Legacy')) {
            $event->addViolation($this, $dependency);
        }
    }

    public function ruleName(): string
    {
        return 'ForbidLegacyDependency';
    }

    public function ruleDescription(): string
    {
        return 'Classes should not depend on legacy classes.';
    }
}
```

### 注册事件订阅者

```yaml
services:
  App\Deptrac\ForbidLegacyDependencyRule:
    tags:
      - { name: 'kernel.event_subscriber' }
```

## 9.5 自定义输出格式化器

### 示例：CSV 格式化器

```php
<?php
namespace App\Deptrac;

use Deptrac\Deptrac\Contract\OutputFormatter\OutputFormatterInterface;
use Deptrac\Deptrac\Contract\Result\OutputResult;
use Symfony\Component\Console\Output\OutputInterface;

class CsvOutputFormatter implements OutputFormatterInterface
{
    public function getName(): string
    {
        return 'csv';
    }

    public function finish(
        OutputResult $result,
        OutputInterface $output,
        array $outputFormatterInput
    ): void {
        $output->writeln('file,line,dependent,dependee,layer_from,layer_to');
        
        foreach ($result->violations() as $violation) {
            $dep = $violation->getDependency();
            $output->writeln(sprintf(
                '%s,%d,%s,%s,%s,%s',
                $dep->getContext()->fileOccurrence->filepath,
                $dep->getContext()->fileOccurrence->line,
                $dep->getDependant()->toString(),
                $dep->getDependee()->toString(),
                $violation->getDependerLayer(),
                $violation->getDependentLayer()
            ));
        }
    }
}
```

### 注册格式化器

```yaml
services:
  App\Deptrac\CsvOutputFormatter:
    tags:
      - { name: 'output_formatter' }
```

### 使用

```bash
vendor/bin/deptrac --formatter=csv --output=violations.csv
```

## 9.6 引用提取器扩展

引用提取器用于从 AST 节点中提取新类型的依赖关系。

### 内置提取器

Deptrac 内置了多种提取器：

| 提取器 | 检测内容 |
|--------|---------|
| `ClassExtractor` | 类定义（继承、实现） |
| `FunctionLikeExtractor` | 函数参数和返回类型 |
| `NewExtractor` | `new` 实例化 |
| `StaticCallExtractor` | 静态方法调用 |
| `InstanceofExtractor` | `instanceof` 检查 |
| `UseExtractor` | `use` 语句 |
| `CatchExtractor` | `catch` 语句 |
| `AnonymousClassExtractor` | 匿名类 |

### 自定义提取器示例

```php
<?php
namespace App\Deptrac;

use Deptrac\Deptrac\Contract\Ast\ReferenceExtractorInterface;
use PhpParser\Node;

class AnnotationExtractor implements ReferenceExtractorInterface
{
    public function processNode(
        Node $node, 
        /* ... 其他参数 */
    ): void {
        // 从自定义注解中提取依赖
        // 比如 @uses App\SomeClass 这样的注解
    }

    public function getNodeType(): string
    {
        return Node\Stmt\Class_::class;
    }
}
```

### 注册提取器

```yaml
services:
  App\Deptrac\AnnotationExtractor:
    tags:
      - { name: 'reference_extractors' }
```

## 9.7 Symfony 服务容器集成

所有扩展都通过 Symfony 的服务容器注册。配置方式有两种：

### YAML 方式

```yaml
# deptrac.yaml
services:
  App\Deptrac\CustomCollector:
    tags:
      - { name: 'collector', type: 'custom' }
  
  App\Deptrac\CustomRule:
    tags:
      - { name: 'kernel.event_subscriber' }
```

### PHP 方式

```php
// deptrac.php
return static function (DeptracConfig $config, ContainerConfigurator $containerConfigurator): void {
    // ... 配置层和规则 ...

    // 注册自定义服务
    $services = $containerConfigurator->services();
    
    $services->set(CustomCollector::class)
        ->tag('collector', ['type' => 'custom']);
    
    $services->set(CustomRule::class)
        ->tag('kernel.event_subscriber');
};
```

## 9.8 Contract 命名空间 — 稳定的 API

所有扩展接口都在 `Deptrac\Deptrac\Contract\` 命名空间下，受向后兼容策略保护：

```
Deptrac\Deptrac\Contract\
├── Analyser\          ← 分析器相关接口
├── Ast\               ← AST 相关接口
│   └── AstMap\        ← AST 映射接口
├── Config\            ← 配置相关接口
│   ├── Collector\     ← 收集器配置
│   └── Formatter\     ← 格式化器配置
├── Dependency\        ← 依赖相关接口
├── Layer\             ← 层相关接口
├── OutputFormatter\   ← 输出格式化器接口
└── Result\            ← 结果相关接口
```

> ⚠️ **重要**：只有 `Contract` 命名空间下的接口是公开 API。
> 其他命名空间下的代码可能在任何版本更新中改变！

## 9.9 PHPStan 解析器（实验性特性）

启用 PHPStan 解析器可以获得更精确的类型推断：

```yaml
deptrac:
  feature_flags:
    phpstan_parser: true
```

**PHPStan 解析器的优势**：
- 更好的泛型类型推断
- 更准确的 Union/Intersection 类型解析
- 更好的 PHPDoc 注解支持

**注意事项**：
- 实验性功能，可能有不稳定行为
- 可能与某些代码模式不兼容
- 分析速度可能略慢

## 9.10 扩展开发最佳实践

### 1. 从 Contract 命名空间出发

```php
// ✅ 好：使用 Contract 命名空间的接口
use Deptrac\Deptrac\Contract\Layer\CollectorInterface;

// ❌ 避免：使用内部命名空间
use Deptrac\Deptrac\Core\Layer\SomeInternalClass;
```

### 2. 测试你的扩展

```php
class CustomCollectorTest extends TestCase
{
    public function testSatisfy(): void
    {
        $collector = new CustomCollector();
        $config = ['param' => 'value'];
        
        // 创建 mock TokenReference
        $reference = $this->createMock(TokenReference::class);
        
        $this->assertTrue($collector->satisfy($config, $reference));
    }
}
```

### 3. 保持扩展简单

```
好的扩展：
✅ 做一件事，做好
✅ 配置灵活
✅ 有清晰的错误提示

不好的扩展：
❌ 功能过于复杂
❌ 硬编码配置
❌ 依赖内部实现
```

## 9.11 本章小结

| 扩展类型 | 适用场景 | 复杂度 |
|---------|---------|--------|
| 自定义收集器 | 新的代码分组方式 | 低 |
| 事件订阅者 | 自定义违规规则 | 中 |
| 输出格式化器 | 新的输出格式 | 低 |
| 引用提取器 | 新的依赖类型检测 | 高 |
| AST 解析器 | 替换默认解析器 | 很高 |

---

**下一章**：[第 10 章：项目生命周期与避坑指南](./10-lifecycle-and-pitfalls.md) — 项目早/中/晚期的策略和常见陷阱！
