# 示例

展示四条原则的真实代码示例。每个示例都展示了 LLM 常犯的错误以及如何修正。

---

## 1. 先思考再编码

### 示例 1：隐藏的假设

**用户请求：** "添加一个导出用户数据的功能"

**❌ LLM 的做法（错误假设）**

```python
def export_users(format='json'):
    """Export all users to a file."""
    users = User.query.all()

    if format == 'json':
        with open('users.json', 'w') as f:
            json.dump([u.to_dict() for u in users], f)
    elif format == 'csv':
        with open('users.csv', 'w') as f:
            writer = csv.DictWriter(f, fieldnames=['id', 'email', 'name'])
            writer.writeheader()
            writer.writerows([u.to_dict() for u in users])

    return f"Exported {len(users)} users"
```

**问题：**
- 假设应该导出所有用户（分页呢？隐私呢？）
- 没有询问就假设了文件位置
- 假设了要包含哪些字段
- 没有检查实际数据结构就假设了 CSV 字段名

**✅ 正确做法（暴露假设）**

```
在实现之前，我需要澄清几点：

1. **范围**：导出所有用户还是筛选后的子集？（涉及隐私）
2. **格式**：你说的"导出"是指：
   - 在浏览器中下载文件？
   - 后台任务完成后邮件通知？
   - 返回数据的 API 端点？
3. **字段**：哪些用户字段？（有些可能是敏感的）
4. **数据量**：通常有多少用户？（影响实现方案）

最简单的方案：添加一个返回分页 JSON 的 API 端点。
如果需要基于文件的导出，还需要更多信息。

你倾向哪种方案？
```

### 示例 2：多种解读

**用户请求：** "让搜索更快"

**❌ LLM 的做法（默默选择）**

```python
# 添加缓存、数据库索引和异步处理
# 却没有问清楚"更快"指的是什么
@lru_cache(maxsize=1000)
async def search(query: str) -> List[Result]:
    # ... 200 行优化代码
```

**✅ 正确做法**

```
"让搜索更快"可能意味着：

1. **更快的响应时间**（从 500ms 降到 100ms 以内）
   - 添加数据库索引
   - 缓存高频查询
   - 预估工作量：2 小时

2. **处理更多并发搜索**（吞吐量）
   - 添加 async/await
   - 连接池
   - 预估工作量：4 小时

3. **更快的感知速度**（用户体验）
   - 立即显示部分结果
   - 渐进式加载
   - 预估工作量：3 小时

当前搜索典型查询耗时约 500ms。
哪个方面对你的场景最重要？
```

---

## 2. 简单优先

### 示例 1：过度抽象

**用户请求：** "添加一个计算折扣的函数"

**❌ LLM 的做法（过度工程化）**

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Protocol, Union
from dataclasses import dataclass

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float:
        pass

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percentage: float):
        self.percentage = percentage

    def calculate(self, amount: float) -> float:
        return amount * (self.percentage / 100)

class FixedDiscount(DiscountStrategy):
    def __init__(self, fixed_amount: float):
        self.fixed_amount = fixed_amount

    def calculate(self, amount: float) -> float:
        return min(self.fixed_amount, amount)

@dataclass
class DiscountConfig:
    strategy: DiscountStrategy
    min_purchase: float = 0.0
    max_discount: float = float('inf')

class DiscountCalculator:
    def __init__(self, config: DiscountConfig):
        self.config = config

    def apply_discount(self, amount: float) -> float:
        if amount < self.config.min_purchase:
            return 0
        discount = self.config.strategy.calculate(amount)
        return min(discount, self.config.max_discount)

# 一个简单的计算需要 30 多行的初始化代码
```

**✅ 正确做法（简单）**

```python
def calculate_discount(amount: float, percent: float) -> float:
    """计算折扣金额。percent 应为 0-100。"""
    return amount * (percent / 100)

# 使用
discount = calculate_discount(100.0, 10.0)  # 减 10 元
```

**何时增加复杂度：** 只有当你确实需要多种折扣类型时。如果后续有这个需求，到时候再重构。

### 示例 2：臆测的功能

**用户请求：** "把用户偏好保存到数据库"

**❌ LLM 的做法（不必要的功能）**

```python
class PreferenceManager:
    def __init__(self, db, cache=None, validator=None):
        self.db = db
        self.cache = cache or InMemoryCache()
        self.validator = validator or DefaultValidator()

    def save(self, user_id: int, prefs: dict,
             merge: bool = True,
             validate: bool = True,
             notify: bool = False) -> bool:
        """保存偏好，支持可选的合并、验证和通知。"""

        if validate:
            errors = self.validator.validate(prefs)
            if errors:
                raise ValidationError(errors)

        existing = self.get(user_id) if merge else {}

        if merge:
            existing.update(prefs)
            prefs = existing

        self.db.execute(
            "UPDATE users SET preferences = ? WHERE id = ?",
            (json.dumps(prefs), user_id)
        )

        if self.cache:
            self.cache.set(f"prefs:{user_id}", prefs)

        if notify:
            self.notify_preference_change(user_id, prefs)

        return True

    def notify_preference_change(self, user_id: int, prefs: dict):
        # 又是 30 行没人要求的功能
        pass
```

**✅ 正确做法（只做被要求的事）**

```python
def save_preferences(db, user_id: int, preferences: dict):
    """保存用户偏好到数据库。"""
    db.execute(
        "UPDATE users SET preferences = ? WHERE id = ?",
        (json.dumps(preferences), user_id)
    )
```

**按需添加：** 缓存（当性能成为问题时）、验证（当出现脏数据时）、合并（当需求出现时）。

---

## 3. 精准修改

### 示例 1：顺手重构

**用户请求：** "修复空邮箱导致验证器崩溃的 bug"

**❌ LLM 的做法（改太多了）**

```diff
  def validate_user(user_data):
-     # Check email format
-     if not user_data.get('email'):
+     """Validate user data."""
+     email = user_data.get('email', '').strip()
+
+     # Validate email
+     if not email:
          raise ValueError("Email required")
-
-     # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email or '.' not in email.split('@')[1]:
          raise ValueError("Invalid email")

-     # Check username
-     if not user_data.get('username'):
+     # Validate username
+     username = user_data.get('username', '').strip()
+     if not username:
          raise ValueError("Username required")
+     if len(username) < 3:
+         raise ValueError("Username too short")
+     if not username.isalnum():
+         raise ValueError("Username must be alphanumeric")

      return True
```

**问题：**
- 超出 bug 修复范围"改进"了邮箱验证
- 添加了没人要求的用户名验证
- 改了注释
- 加了文档字符串

**✅ 正确做法（精准）**

```diff
  def validate_user(user_data):
      # Check email format
-     if not user_data.get('email'):
+     email = user_data.get('email', '')
+     if not email or not email.strip():
          raise ValueError("Email required")

      # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email:
          raise ValueError("Invalid email")

      # Check username
      if not user_data.get('username'):
          raise ValueError("Username required")

      return True
```

**只改了：** 修复空邮箱处理的那几行。

### 示例 2：风格漂移

**用户请求：** "给上传函数添加日志"

**❌ LLM 的做法（顺手重新格式化）**

```diff
- def upload_file(file_path, destination):
+ def upload_file(file_path: str, destination: str) -> bool:
+     """Upload file to destination with logging."""
+     logger.info(f"Uploading {file_path} to {destination}")
+
      try:
-         with open(file_path, 'rb') as f:
-             data = f.read()
-
-         response = requests.post(destination, files={'file': data})
-
-         if response.status_code == 200:
-             return True
-         else:
-             return False
+         with open(file_path, "rb") as f:
+             data = f.read()
+         response = requests.post(destination, files={"file": data})
+         success = response.status_code == 200
+
+         if success:
+             logger.info(f"Upload successful: {file_path}")
+         else:
+             logger.error(f"Upload failed: {file_path}, status={response.status_code}")
+
+         return success
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f"Upload error for {file_path}: {e}")
          return False
```

**问题：**
- 改了引号风格（'' 变成 ""）
- 加了没人要求的类型注解
- 加了文档字符串
- 重新格式化了空白
- 改了布尔返回逻辑

**✅ 正确做法（匹配现有风格）**

```diff
+ import logging
+
+ logger = logging.getLogger(__name__)
+
  def upload_file(file_path, destination):
+     logger.info(f'Starting upload: {file_path}')
      try:
          with open(file_path, 'rb') as f:
              data = f.read()

          response = requests.post(destination, files={'file': data})

          if response.status_code == 200:
+             logger.info(f'Upload successful: {file_path}')
              return True
          else:
+             logger.error(f'Upload failed: {file_path}, status={response.status_code}')
              return False
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f'Upload error: {file_path}')
          return False
```

**保持一致：** 单引号、无类型注解、现有的布尔返回模式、缩进风格。

---

## 4. 目标驱动执行

### 示例 1：模糊 vs. 可验证

**用户请求：** "修复认证系统"

**❌ LLM 的做法（模糊方案）**

```
我会通过以下步骤修复认证系统：
1. 审查代码
2. 找出问题
3. 做出改进
4. 测试更改

[然后在没有明确成功标准的情况下开始改代码]
```

**✅ 正确做法（可验证的目标）**

```
要"修复认证"，我需要先定义成功标准。
具体要解决什么问题？

如果问题是"修改密码后用户仍保持登录状态"：

计划：
1. 写测试：修改密码 → 验证旧会话已失效
   验证：测试失败（复现 bug）

2. 实现：密码修改时使会话失效
   验证：测试通过

3. 检查边界情况：多个活跃会话、并发修改
   验证：额外测试通过

4. 验证无回归：现有认证测试仍然通过
   验证：完整测试套件全绿

当前认证测试覆盖率：[显示覆盖率]
你遇到的具体认证问题是什么？
```

### 示例 2：带验证的多步骤

**用户请求：** "给 API 添加限流"

**❌ LLM 的做法（一步到位）**

```python
# 在一次 300 行的提交中实现了完整的限流：
# Redis、多种策略、配置系统和监控
# 没有清晰的验证步骤
```

**✅ 正确做法（增量实现，逐步验证）**

```
限流实现计划：

1. 添加基本的内存限流（单个端点）
   验证：
   - 测试：100 个请求 → 前 10 个成功，其余返回 429
   - 手动：curl 端点 11 次，看到限流错误

2. 提取为中间件（应用到所有端点）
   验证：
   - 测试：限流对 /users 和 /posts 都生效
   - 现有端点测试仍然通过

3. 添加 Redis 后端（支持多服务器）
   验证：
   - 测试：限流在应用重启后仍然有效
   - 测试：两个应用实例共享限流计数器

4. 添加配置（每个端点不同的限流）
   验证：
   - 测试：/search 允许 10 次/分钟，/users 允许 100 次/分钟
   - 配置文件正确解析

每一步都可以独立验证和部署。
从第 1 步开始？
```

### 示例 3：测试先行验证

**用户请求：** "有重复分数时排序会出错"

**❌ LLM 的做法（不复现就修复）**

```python
# 没有确认 bug 就直接改排序逻辑
def sort_scores(scores):
    return sorted(scores, key=lambda x: (-x['score'], x['name']))
```

**✅ 正确做法（先复现）**

```python
# 1. 首先，写一个复现问题的测试
def test_sort_with_duplicate_scores():
    """测试多个条目分数相同时的排序。"""
    scores = [
        {'name': 'Alice', 'score': 100},
        {'name': 'Bob', 'score': 100},
        {'name': 'Charlie', 'score': 90},
    ]

    result = sort_scores(scores)

    # bug：重复分数的排序是不确定的
    # 多次运行这个测试，结果应该一致
    assert result[0]['score'] == 100
    assert result[1]['score'] == 100
    assert result[2]['score'] == 90

# 验证：运行测试 10 次 → 排序不一致导致失败

# 2. 现在用稳定排序修复
def sort_scores(scores):
    """按分数降序排列，分数相同时按名字升序。"""
    return sorted(scores, key=lambda x: (-x['score'], x['name']))

# 验证：测试一致通过
```

---

## 反模式总结

| 原则 | 反模式 | 修正 |
|------|--------|------|
| 先思考再编码 | 默默假设文件格式、字段、范围 | 明确列出假设，请求澄清 |
| 简单优先 | 为单个折扣计算使用策略模式 | 一个函数就够，等真正需要时再加复杂度 |
| 精准修改 | 修 bug 时顺手改引号风格、加类型注解 | 只改修复问题的那几行 |
| 目标驱动 | "我会审查并改进代码" | "为 bug X 写测试 → 让测试通过 → 验证无回归" |

## 核心洞察

那些"过度复杂"的示例并非明显错误——它们遵循了设计模式和最佳实践。问题在于**时机**：在需要之前就添加复杂度，这会导致：

- 代码更难理解
- 引入更多 bug
- 实现耗时更长
- 更难测试

而"简单"的版本：
- 更容易理解
- 更快实现
- 更容易测试
- 真正需要时可以再重构

**好代码是简单地解决今天的问题，而不是过早地解决明天的问题。**
