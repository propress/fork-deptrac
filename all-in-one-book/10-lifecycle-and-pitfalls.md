# 第 10 章：项目生命周期与避坑指南

> 🟢🟡🔴 适合阶段：全阶段
> 
> 阅读时间：约 15 分钟

---

## 10.1 项目生命周期中的 Deptrac

```
项目生命周期          Deptrac 的角色                       关键行动
────────────────────────────────────────────────────────────────────

项目启动 (第0天)       预防性守护                          定义理想架构
     │                                                    从第一天就引入
     ▼
快速开发期 (0-6月)     实时监控                            CI/CD 自动检查
     │                 "在问题变大之前发现它"                适度调整规则
     ▼
稳定迭代期 (6-18月)    持续守护                            收紧规则
     │                 "确保新代码不破坏架构"                消除 baseline
     ▼
成熟维护期 (18月+)     架构文档化                          可视化架构图
     │                 "新人通过 Deptrac 理解架构"           用于团队知识传递
     ▼
重构/拆分期            验证工具                             验证模块独立性
                       "确认拆分的可行性"                   指导微服务拆分
```

## 10.2 项目早期策略

### 新项目从 Day 1 引入

```bash
# 项目初始化时就添加 Deptrac
composer require --dev deptrac/deptrac
vendor/bin/deptrac init
```

### 早期配置建议

```yaml
# deptrac.yaml - 新项目初始配置
deptrac:
  paths:
    - ./src

  layers:
    # 早期层定义不需要太多，3-5 个就够
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

  ruleset:
    Controller:
      - Service
    Service:
      - Repository
    Repository: ~
```

### 早期常见坑

#### 坑 1：过度设计架构层

```
❌ 早期就定义 20+ 个层
   → 维护成本高
   → 频繁调整
   → 团队抵触

✅ 从 3-5 个核心层开始
   → 随项目增长逐步添加
   → 团队容易接受
```

#### 坑 2：规则太严格

```
❌ 一开始就 --fail-on-uncovered
   → 每添加一个新工具类就报错
   → 开发效率受影响

✅ 先只检查层间违规
   → 未覆盖依赖先不管
   → 等架构稳定后再收紧
```

#### 坑 3：没有团队共识

```
❌ 一个人悄悄加了 Deptrac 和规则
   → 其他人不理解为什么 CI 失败
   → 导致绕过或删除

✅ 先和团队讨论架构规则
   → 达成共识后再用 Deptrac 实施
   → 大家理解为什么需要这些规则
```

## 10.3 项目中期策略

### 已有项目引入 Deptrac

对于已经有大量代码的项目，用 **Baseline** 机制渐进式引入：

```
Day 1:  分析现状 → 生成 baseline → CI 集成
        ┌──────────────────────────────┐
        │ baseline 中有 87 个违规       │
        │ 新代码：0 个违规              │
        │ CI：✅ 通过                   │
        └──────────────────────────────┘

Week 2: 修复 5 个简单的违规
        ┌──────────────────────────────┐
        │ baseline 中有 82 个违规       │
        │ 新代码：0 个违规              │
        │ CI：✅ 通过                   │
        └──────────────────────────────┘

Month 2: 持续修复
        ┌──────────────────────────────┐
        │ baseline 中有 45 个违规       │
        │ 新代码：0 个违规              │
        │ CI：✅ 通过                   │
        └──────────────────────────────┘

Month 6: 清除完毕
        ┌──────────────────────────────┐
        │ baseline 中有 0 个违规 🎉    │
        │ 新代码：0 个违规              │
        │ CI：✅ 通过                   │
        └──────────────────────────────┘
```

### 操作步骤

```bash
# 1. 先运行看看有多少违规
vendor/bin/deptrac

# 2. 生成 baseline
vendor/bin/deptrac --formatter=baseline --output=deptrac.baseline.yaml

# 3. 导入 baseline
# 编辑 deptrac.yaml 添加 imports:
#   imports:
#     - deptrac.baseline.yaml

# 4. 提交到版本控制
git add deptrac.yaml deptrac.baseline.yaml
git commit -m "chore: introduce deptrac with baseline"

# 5. 开始逐步修复
# 每次修复后更新 baseline
vendor/bin/deptrac --formatter=baseline --output=deptrac.baseline.yaml
```

### 中期常见坑

#### 坑 4：Baseline 只增不减

```
❌ 只在出现新违规时加入 baseline，但从不修复老的
   → baseline 越来越大
   → 失去了工具的意义

✅ 设定目标：每个 Sprint 修复 N 个 baseline 违规
   → 比如每两周修 5 个
   → 持续改善，可视化进度
```

#### 坑 5：层的划分不合理导致大量违规

```
❌ 按命名约定分层，但代码不遵循命名约定
   例：有的 Service 叫 Manager、Handler、Processor

✅ 结合 directory + classLike
   或者统一命名规范后再分层
```

#### 坑 6：忘记在 CI 中启用

```
❌ 本地能跑 Deptrac，但 CI 中没有
   → 部分开发者不跑 Deptrac

✅ CI 中必须有 Deptrac 检查
   → 而且要设为 required check（阻止合并）
```

## 10.4 项目晚期/成熟期策略

### 收紧规则

```yaml
# 成熟项目的严格配置
deptrac:
  analyser:
    types:
      - class
      - function
      - function_call
      - superglobal        # 也监控超全局变量

  # 所有层都不应该有未覆盖的依赖
  # 使用 --fail-on-uncovered
```

```bash
# CI 中使用严格模式
vendor/bin/deptrac \
  --no-progress \
  --report-uncovered \
  --fail-on-uncovered \
  --report-skipped
```

### 架构文档化

使用 Deptrac 自动生成架构文档：

```bash
# 生成 Mermaid 图表嵌入文档
vendor/bin/deptrac --formatter=mermaidjs --output=docs/architecture.mmd

# 生成架构图片
vendor/bin/deptrac --formatter=graphviz-image --output=docs/architecture.png
```

### 为微服务拆分做准备

```yaml
# 验证模块独立性
deptrac:
  layers:
    - name: UserModule
      collectors:
        - type: directory
          value: src/User/.*
    - name: OrderModule
      collectors:
        - type: directory
          value: src/Order/.*

  ruleset:
    UserModule: ~           # 如果这里显示 0 violations
    OrderModule: ~          # 说明两个模块可以安全拆分
```

## 10.5 不同阶段的坑总览

### 🟢 新手阶段

| # | 坑 | 症状 | 解决方案 |
|---|---|------|---------|
| 1 | 正则写错 | 层里没有匹配到类 | 用 `debug:layer` 验证 |
| 2 | 忘记默认禁止原则 | 意外的违规 | 仔细检查 ruleset |
| 3 | FQCN 不理解 | 正则匹配不到 | 理解全限定类名 |
| 4 | YAML 格式错误 | 启动就报错 | 注意缩进和语法 |
| 5 | 找不到配置文件 | "No config file found" | 检查文件名和 `-c` 参数 |

### 🟡 熟悉阶段

| # | 坑 | 症状 | 解决方案 |
|---|---|------|---------|
| 6 | imports 放错位置 | 配置不生效 | imports 在顶层 |
| 7 | 多收集器是 OR 关系 | 匹配到意外的类 | 用 bool 实现 AND |
| 8 | 缓存过期 | 改了代码但结果没变 | `--clear-cache` |
| 9 | 排除文件格式 | 文件没被排除 | 用正则分隔符 `#regex#` |
| 10 | PHP 配置缺 use 语句 | "Class not found" | 添加正确的 use |

### 🔴 精通阶段

| # | 坑 | 症状 | 解决方案 |
|---|---|------|---------|
| 11 | 扩展用了内部 API | 升级后扩展崩溃 | 只用 Contract 命名空间 |
| 12 | 过多的层导致规则爆炸 | 维护困难 | 使用 PHP 配置动态生成 |
| 13 | 传递依赖理解错误 | `+` 前缀的意外行为 | 仔细理解传递规则 |
| 14 | 事件订阅者优先级 | 自定义规则不生效 | 检查事件优先级 |
| 15 | Baseline 冲突 | 合并后 baseline 不一致 | 重新生成 baseline |

## 10.6 团队推广指南

### 如何让团队接受 Deptrac

```
第 1 步：先给团队展示问题
         "看，我们的代码有 87 个架构违规"
         
第 2 步：展示解决方案
         "Deptrac 可以自动检测这些问题"
         
第 3 步：展示不影响现有工作
         "用 baseline，现有代码不受影响，只检查新代码"
         
第 4 步：从小处开始
         "先保护最核心的 3 个层"
         
第 5 步：持续展示成果
         "上个月我们修复了 12 个架构违规，baseline 从 87 降到 75"
```

### 团队沟通模板

```
📣 团队通知：我们引入了 Deptrac 架构检查

为什么？
- 防止架构腐化（Controller 直接调 Repository 这类问题）
- 自动化检查，不依赖人工 Review

影响是什么？
- CI 中新增了架构检查步骤
- 现有代码不受影响（已通过 baseline 跳过）
- 新代码必须遵循架构规则

你需要做什么？
- 如果 CI 报 deptrac 错误，说明新代码违反了架构规则
- 修改代码消除违规（不是改配置！）
- 如果不确定如何修复，找 [架构负责人] 讨论

常见问题：
Q: 这会影响开发速度吗？
A: 检查只需几秒钟，比跑测试快得多

Q: 如果我认为规则不合理怎么办？
A: 提 Issue 讨论，我们可以调整规则
```

## 10.7 版本升级指南

### 从 0.x 升级到 1.x

关键变化：
- 配置文件默认名从 `depfile.yaml` 改为 `deptrac.yaml`
- 配置需要嵌套在 `deptrac:` 下
- `className` 收集器改为 `classLike`
- `%depfileDirectory%` 改为 `%projectDirectory%`

### 从 1.x 升级到 2.x

关键变化：
- PHP 最低版本要求 8.2
- 一些内部接口签名变化（只影响扩展开发者）
- 默认依赖类型有变化

### 升级最佳实践

```bash
# 1. 升级前先记录当前状态
vendor/bin/deptrac --formatter=json --output=before-upgrade.json

# 2. 升级
composer update deptrac/deptrac

# 3. 清除缓存
vendor/bin/deptrac --clear-cache

# 4. 运行检查
vendor/bin/deptrac

# 5. 对比结果
vendor/bin/deptrac --formatter=json --output=after-upgrade.json
# 对比两个 JSON 文件，确认差异是预期的
```

## 10.8 性能优化建议

| 阶段 | 优化措施 |
|------|---------|
| 小项目 (<100 类) | 默认配置即可 |
| 中项目 (100-1000 类) | 启用缓存 + 精确 paths |
| 大项目 (1000+ 类) | 缓存 + 排除不必要的文件 + PHP 配置 |
| 超大项目 (10000+ 类) | 考虑分模块配置 + CI 缓存 |

```bash
# 大项目性能优化示例
vendor/bin/deptrac \
  --cache-file=.cache/deptrac.cache \  # 指定缓存位置
  --no-progress \                       # 禁用进度条（CI）
  -c deptrac.php                        # 使用 PHP 配置（更快解析）
```

## 10.9 本章小结

| 阶段 | 关键策略 | 常见坑 |
|------|---------|--------|
| 早期 | 3-5 个层，简单规则 | 过度设计、规则太严 |
| 中期 | Baseline 渐进式引入 | Baseline 只增不减 |
| 晚期 | 收紧规则、文档化 | 忽视维护 |
| 全程 | CI 集成是必须的 | 没有团队共识 |

---

**附录**：[Pro 版对比、FAQ、速查表](./appendix.md) — 常见问题和快速参考！
