# pytest 学习笔记：Fixtures (固件)

## 1. 什么是 Fixtures？

Fixture 是为测试提供**上下文环境**的函数。它可以用来：

- 准备测试数据（如：创建临时文件、初始化数据库）。
- 实例化对象（如：启动浏览器驱动、连接 API）。
- 清理资源（如：关闭连接、删除文件）。

## 2. 基本定义与引用

使用 `@pytest.fixture` 装饰器定义，并在测试函数的参数中直接传入函数名来“注入”依赖。

```python
import pytest

# 定义fixture
@pytest.fixture(scpoe="xx", autouse=False)
def db_connection():
    # 准备工作
    conn = {"status": "connected", "user": "admin"}
    return conn

# 使用fixture
def test_query(  ):
    # 直接使用 fixture 的返回值
    assert db_connection["user"] == "admin"]
```

## 3. 作用域 (Scope)

Fixture 的生命周期可以通过 `scope` 参数控制，避免不必要的重复初始化：

| **作用域 (Scope)** | **执行频率**          | **适用场景**   |
| --------------- | ----------------- | ---------- |
| `function` (默认) | **每个测试函数**执行一次    | 独立数据环境     |
| `class`         | **每个测试类**执行一次     | 类内共享资源     |
| `module`        | **每个 .py 文件**执行一次 | 模块级初始化     |
| `session`       | **整个测试会话**只执行一次   | 全局配置、数据库连接 |

## 4. 清理逻辑：`yield` 关键字

如果你需要在测试完成后执行清理操作（如关闭数据库），请使用 `yield` 代替 `return`。

```Python
@pytest.fixture
def smtp_connection():
    # Setup: 执行测试前的操作
    smtp = "Connecting..."
    yield smtp  # 暂停并把控制权交给测试函数
    # Teardown: 执行测试后的清理
    print("Closing connection")
```

## 5. 共享 Fixture：`conftest.py`

如果你希望多个测试文件共享同一个 Fixture，无需重复导入。只需在项目根目录或子目录创建 `conftest.py` 文件：

