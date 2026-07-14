# pytest 学习笔记：标记 (Marks)

## 1. 什么是 Marks？

Marks 允许你为测试用例添加元数据。你可以通过标记来控制测试的执行逻辑（如跳过某些测试）或对测试进行逻辑分组（如区分冒烟测试和回归测试）。

## 2. 内置标记 (Built-in Marks)

pytest 提供了几个非常实用的内置标记：

| **标记**                     | **作用**    | **示例**                                                           |
| -------------------------- | --------- | ---------------------------------------------------------------- |
| `@pytest.mark.skip`        | 始终跳过该测试   | `@pytest.mark.skip(reason="尚未开发完成")`                             |
| `@pytest.mark.skipif`      | 满足特定条件时跳过 | `@pytest.mark.skipif(sys.platform == 'win32', reason="不在Win运行")` |
| `@pytest.mark.xfail`       | 预期该测试会失败  | `@pytest.mark.xfail(reason="已知Bug待修复")`                          |
| `@pytest.mark.parametrize` | 参数化测试     | _（已在主题二详细记录）_                                                    |

## 3. 自定义标记 (Custom Marks)

你可以根据业务需求自定义标签，以便过滤运行。
### 注册自定义标记

为了避免拼写错误或警告，建议在 `pytest.ini` 文件中注册你的自定义标记：

```Ini
# pytest.ini
[pytest]
markers =
    smoke: 冒烟测试用例
    regression: 回归测试用例
```

### 使用方法

```Python
import pytest

@pytest.mark.smoke
def test_login_process():
    assert True

@pytest.mark.regression
def test_complex_calculation():
    assert 1 + 1 == 2
```

### 运行特定标记的测试

在命令行中使用 `-m` 参数：

```Bash
# 只运行标记为 smoke 的测试
pytest -m smoke

# 运行标记为 smoke 且不含 regression 的测试
pytest -m "smoke and not regression"
```


---

# 参数化测试

## 1. 为什么需要参数化？

如果你有 10 组输入数据需要测试同一个函数，传统的写法会产生大量冗余代码。参数化可以将**测试逻辑**与**测试数据**分离。

## 2. 核心语法：`@pytest.mark.parametrize`

通过装饰器，你可以将数据注入到测试函数的参数中。

### 基础用法

```python
import pytest

def add(a, b):
    return a + b

# 第一个参数是字符串，定义参数名
# 第二个参数是列表/元组，定义具体的数据集
@pytest.mark.parametrize("x, y, expected", [
    (1, 1, 2),
    (2, 3, 5),
    (10, 20, 30),
])
def test_add(x, y, expected):
    assert add(x, y) == expected
```

## 3. 进阶用法

### 组合参数化 (笛卡尔积)

如果你叠加使用装饰器，pytest 会自动组合所有可能的参数：

```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_foo(x, y):
    # 此测试会运行 4 次：(0,2), (0,3), (1,2), (1,3)
    pass
```

### 为参数添加标签 (ids)

当数据量很大时，通过 `ids` 参数可以为每一组数据命名，方便在测试报告中识别：

```python
@pytest.mark.parametrize(
    "user, secret", 
    [("admin", "123"), ("guest", "000")],
    ids=["管理员登录", "访客登录"]
)
def test_login(user, secret):
    pass
```