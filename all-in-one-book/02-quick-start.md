# 第 2 章：5 分钟快速上手

> 🟢 适合阶段：完全新手
> 
> 阅读时间：约 10 分钟（含动手实操）

---

## 2.1 安装 Deptrac

有三种安装方式，推荐从 **方式 1** 开始：

### 方式 1：通过 Composer 安装（推荐）

```bash
# 在你的 PHP 项目根目录下
composer require --dev deptrac/deptrac
```

安装后可执行文件位于 `vendor/bin/deptrac`。

> ⚠️ **新手常见坑 #1**：如果安装报依赖冲突（尤其是 Symfony 版本不兼容），
> 参见下方"安装遇到冲突怎么办"。

### 方式 2：下载 PHAR 文件

```bash
# 从 GitHub Release 页面下载最新 phar
curl -LO https://github.com/deptrac/deptrac/releases/latest/download/deptrac.phar

# 给执行权限
chmod +x deptrac.phar

# 可选：移动到全局可用
sudo mv deptrac.phar /usr/local/bin/deptrac
```

### 方式 3：通过 PHIVE 安装

```bash
phive install -g deptrac/deptrac
```

### 安装遇到 Composer 依赖冲突怎么办？

当你的项目依赖的 Symfony 版本与 Deptrac 不兼容时，使用 `bamarni/composer-bin-plugin` 隔离安装：

```bash
# 1. 安装插件
composer require --dev bamarni/composer-bin-plugin

# 2. 在隔离环境中安装 Deptrac
composer bin deptrac require --dev deptrac/deptrac

# 3. 运行 Deptrac（路径较长）
vendor-bin/deptrac/vendor/deptrac/deptrac/deptrac
```

> 💡 **小技巧**：在 `composer.json` 的 `scripts` 中添加快捷方式：
> ```json
> {
>   "scripts": {
>     "deptrac": "vendor-bin/deptrac/vendor/deptrac/deptrac/deptrac"
>   }
> }
> ```
> 然后就可以用 `composer deptrac` 来运行了。

### 验证安装

```bash
# 如果通过 Composer 安装
vendor/bin/deptrac --version

# 如果通过 PHAR 安装
php deptrac.phar --version

# 输出类似：
# deptrac 2.x.x
```

## 2.2 初始化配置文件

```bash
vendor/bin/deptrac init
```

这会在项目根目录生成 `deptrac.yaml` 文件，内容如下：

```yaml
deptrac:
  paths:
    - ./src
  exclude_files:
    - '#.*test.*#'
  layers:
    - name: Controller
      collectors:
        - type: classLike
          value: .*Controller.*
    - name: Repository
      collectors:
        - type: classLike
          value: .*Repository.*
    - name: Service
      collectors:
        - type: classLike
          value: .*Service.*
  ruleset:
    Controller:
      - Service
    Service:
      - Repository
    Repository: ~
```

## 2.3 理解配置文件（逐行解读）

让我们一行一行理解这个配置：

```yaml
deptrac:
  # ① 告诉 Deptrac 去哪里扫描代码
  paths:
    - ./src                    # 扫描 src 目录下的所有 PHP 文件

  # ② 排除不需要分析的文件
  exclude_files:
    - '#.*test.*#'             # 排除测试文件（正则表达式）

  # ③ 定义架构层（最核心的部分！）
  layers:
    - name: Controller         # 层名称
      collectors:              # 收集器：决定哪些类属于这一层
        - type: classLike      # 收集器类型：匹配类/接口/trait/enum
          value: .*Controller.*  # 正则：类名包含 "Controller" 的都属于这层

    - name: Repository
      collectors:
        - type: classLike
          value: .*Repository.*

    - name: Service
      collectors:
        - type: classLike
          value: .*Service.*

  # ④ 定义依赖规则
  ruleset:
    Controller:                # Controller 层允许依赖的层：
      - Service                # ✅ 可以依赖 Service
                               # ❌ 不能依赖 Repository（没列出 = 禁止）
    Service:                   # Service 层允许依赖的层：
      - Repository             # ✅ 可以依赖 Repository
                               # ❌ 不能依赖 Controller（没列出 = 禁止）
    Repository: ~              # Repository 层：不允许依赖任何其他层
                               # ~ 是 YAML 的 null，表示空列表
```

> 🔑 **核心规则**：**没有显式允许的依赖 = 默认禁止！**
> 
> 这是 Deptrac 最重要的原则。

用图来表示这个规则：

```
     Controller
         │
         │ ✅ 允许
         ▼
      Service        Controller ──✖──▶ Repository  (禁止!)
         │
         │ ✅ 允许
         ▼
     Repository      Repository ──✖──▶ Service     (禁止!)
                     Repository ──✖──▶ Controller  (禁止!)
```

## 2.4 运行第一次分析

```bash
vendor/bin/deptrac

# 等价于：
vendor/bin/deptrac analyse --config-file=deptrac.yaml
```

### 情况 A：一切正常 🎉

```
 ----------- --------- ----------- --------- ---------- -------- --------
  Reason      Dep.      Dep.on      Rule      Skipped    Uncov.   Allowed
 ----------- --------- ----------- --------- ---------- -------- --------
                                                                  
 ----------- --------- ----------- --------- ---------- -------- --------
  0           0         0           0         0          0        15
 ----------- --------- ----------- --------- ---------- -------- --------
```

命令返回退出码 `0` —— 你的架构是干净的！

### 情况 B：发现违规 ❌

```
 src/Controller/UserController.php::12
     UserController must not depend on UserRepository (Controller on Repository)

 ----------- ------------ -------------- --------- ---------- -------- --------
  Reason      Dep.         Dep.on         Rule      Skipped    Uncov.   Allowed
 ----------- ------------ -------------- --------- ---------- -------- --------
  Violation   Controller   Repository     1         0          0        14
 ----------- ------------ -------------- --------- ---------- -------- --------

 [ERROR] Analysis finished with 1 violations.
```

命令返回退出码 `1` —— 有架构违规需要修复！

### 输出结果表格解读

| 列名 | 含义 |
|------|------|
| **Reason** | 违规的类型 |
| **Dep.** | 依赖方（哪个层的代码产生了依赖） |
| **Dep.on** | 被依赖方（依赖了哪个层） |
| **Rule** | 违反的规则数量 |
| **Skipped** | 被跳过的已知违规数量（通过 baseline） |
| **Uncov.** | 未覆盖的依赖（代码没有归属到任何层） |
| **Allowed** | 允许的依赖数量 |

## 2.5 来，动手做一个完整的例子！

让我们创建一个最小化的项目来亲自体验：

### 第 1 步：创建项目结构

```bash
mkdir deptrac-demo && cd deptrac-demo
composer init --name=demo/deptrac-demo --type=project -n
composer require --dev deptrac/deptrac
mkdir -p src/{Controller,Service,Repository}
```

### 第 2 步：创建几个示例类

**src/Controller/UserController.php**
```php
<?php
namespace Demo\Controller;

use Demo\Service\UserService;

class UserController
{
    public function __construct(private UserService $service) {}
    
    public function list(): array
    {
        return $this->service->getAllUsers();
    }
}
```

**src/Service/UserService.php**
```php
<?php
namespace Demo\Service;

use Demo\Repository\UserRepository;

class UserService
{
    public function __construct(private UserRepository $repo) {}
    
    public function getAllUsers(): array
    {
        return $this->repo->findAll();
    }
}
```

**src/Repository/UserRepository.php**
```php
<?php
namespace Demo\Repository;

class UserRepository
{
    public function findAll(): array
    {
        return []; // 简化示例
    }
}
```

### 第 3 步：配置 composer.json 的 autoload

在 `composer.json` 中添加：
```json
{
    "autoload": {
        "psr-4": {
            "Demo\\": "src/"
        }
    }
}
```

然后运行 `composer dump-autoload`。

### 第 4 步：创建 deptrac.yaml

```yaml
deptrac:
  paths:
    - ./src
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
  ruleset:
    Controller:
      - Service
    Service:
      - Repository
    Repository: ~
```

### 第 5 步：运行分析

```bash
vendor/bin/deptrac
```

**预期结果**：0 violations ✅ — 因为我们的代码严格遵守了架构规则。

### 第 6 步：故意制造一个违规

修改 `src/Controller/UserController.php`，直接依赖 Repository：

```php
<?php
namespace Demo\Controller;

use Demo\Service\UserService;
use Demo\Repository\UserRepository;  // ← 新增：直接依赖 Repository

class UserController
{
    public function __construct(
        private UserService $service,
        private UserRepository $repo  // ← 违规！
    ) {}
}
```

再次运行：

```bash
vendor/bin/deptrac
```

**预期结果**：1 violation ❌ — Deptrac 抓到了这个架构违规！

```
 src/Controller/UserController.php::11
     UserController must not depend on UserRepository (Controller on Repository)
```

🎉 **恭喜！你已经掌握了 Deptrac 的基本用法！**

## 2.6 可选：安装 Graphviz 生成架构图

```bash
# macOS
brew install graphviz

# Ubuntu / Debian
sudo apt-get install graphviz

# Windows
# 从 https://graphviz.gitlab.io/_pages/Download/Download_windows.html 下载
```

安装后可以生成可视化的架构依赖图：

```bash
# 生成图片文件
vendor/bin/deptrac --formatter=graphviz-image --output=architecture.png

# 直接打开（macOS/Linux 桌面环境）
vendor/bin/deptrac --formatter=graphviz-display
```

## 2.7 常用命令速查

| 命令 | 作用 |
|------|------|
| `vendor/bin/deptrac` | 运行分析（默认命令） |
| `vendor/bin/deptrac init` | 生成默认配置文件 |
| `vendor/bin/deptrac -v` | 显示详细输出 |
| `vendor/bin/deptrac --no-cache` | 禁用缓存重新分析 |
| `vendor/bin/deptrac --clear-cache` | 清除缓存后分析 |
| `vendor/bin/deptrac -c my-config.yaml` | 指定配置文件 |
| `vendor/bin/deptrac --formatter=json` | 以 JSON 格式输出 |

## 2.8 新手阶段常见坑

### 坑 1：正则表达式写错了，层里一个类都没有

**症状**：运行后 0 violations 但也 0 allowed，好像什么都没分析。

**排查**：
```bash
# 检查某个层匹配到了哪些类
vendor/bin/deptrac debug:layer Controller
```

如果输出为空，说明正则没匹配到任何类。检查你的 `value` 正则是否正确。

### 坑 2：搞混了 `classLike` 的正则格式

`classLike` 收集器的 `value` 是 **不带分隔符** 的正则表达式（Deptrac 会自动加上 `/YOUR_VALUE/i`）。

```yaml
# ✅ 正确
- type: classLike
  value: .*Controller.*

# ❌ 错误 — 不需要加分隔符
- type: classLike
  value: /.*Controller.*/i

# ✅ 如果你需要完全控制正则，使用 classNameRegex
- type: classNameRegex
  value: '#.*Controller.*#i'
```

### 坑 3：忘记了 "默认禁止" 原则

新手常犯的错：定义了层和 ruleset，但是 ruleset 里漏掉了某个层，导致那个层"不允许依赖任何层"。

```yaml
ruleset:
  Controller:
    - Service
  Service:
    - Repository
  # 忘了写 Repository: ~ ← 这个倒是问题不大，因为默认就是不允许依赖
  # 但如果你的 Entity 层需要被多个层依赖，别忘了在每个需要的地方加上！
```

### 坑 4：类名不完整或命名空间不匹配

Deptrac 匹配的是 **全限定类名（FQCN）**。比如 `App\Controller\UserController`，你需要确保正则能匹配整个字符串。

```yaml
# ✅ 能匹配 App\Controller\UserController
- type: classLike
  value: .*Controller.*

# ❌ 只能匹配到 "UserController"，但 FQCN 是 "App\Controller\UserController"
#    实际上 .*Controller.* 也能匹配上面的，因为正则是部分匹配
#    但如果你写 ^Controller，就只匹配开头是 Controller 的类
- type: classLike
  value: ^Controller
```

## 2.9 本章小结

| 完成了 | 学到了 |
|--------|--------|
| ✅ 安装 Deptrac | 3 种安装方式 + 冲突解决方案 |
| ✅ 生成配置文件 | `deptrac init` 命令 |
| ✅ 理解配置结构 | paths / layers / collectors / ruleset |
| ✅ 运行分析 | `deptrac analyse` 命令和结果解读 |
| ✅ 动手实验 | 创建 demo 项目并检测违规 |
| ✅ 了解常见坑 | 正则、默认禁止原则、FQCN |

---

**下一章**：[第 3 章：核心概念深入理解](./03-core-concepts.md) — 深入理解 Layer、Collector、Ruleset、Violation 的工作原理！
