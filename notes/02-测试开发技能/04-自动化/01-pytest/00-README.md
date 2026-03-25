# pytest 学习笔记：基础篇

## 1. 环境验证

在开始之前，确保你的环境中已安装 pytest：

Bash

```
pip install pytest
pytest --version
```

## 2. 标准命名约定

pytest 通过特定的命名模式来自动识别测试文件和函数。**只有符合以下规则的内容才会被执行：**

- **模块 (文件)**: `test_*.py` 或 `*_test.py`
    
- **类 (Class)**: `Test*` (注意：类中不能包含 `__init__` 方法)
    
- **函数/方法**: `test_*`
    

## 3. 断言 (Assertions)

pytest 允许使用 Python 标准的 `assert` 关键字，无需学习复杂的 API。

Python

```python
def test_string_check():
    name = "pytest"
    assert "py" in name

def test_list_comparison():
    assert [1, 2] == [1, 2]
```

## 4. 运行测试常用参数

在终端执行时，这些参数能大幅提升你的调试效率：

| **参数**        | **说明**                            | **示例**               |
| ------------- | --------------------------------- | -------------------- |
| `-v`          | **Verbose**: 详细输出，显示每个测试函数的名称和结果。 | `pytest -v`          |
| `-q`          | **Quiet**: 静默模式，只显示简单的进度（点号）。     | `pytest -q`          |
| `-k`          | **Keyword**: 只运行名称包含特定关键字的测试。     | `pytest -k "login"`  |
| `-x`          | **Exit**: 遇到第一个失败 (Fail) 立即停止运行。  | `pytest -x`          |
| `--maxfail=n` | 设定失败 $n$ 次后停止运行。                  | `pytest --maxfail=2` |
