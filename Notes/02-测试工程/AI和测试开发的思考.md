部分内容贴自AI对话


如果你的目标是**测试开发（Test Development / [[SDET]]）**，那么 AI 不应该只是帮你「写 自动化脚本」，而应该成为整个测试工程体系的一部分。


# 1. AI 辅助测试
AI 不负责执行测试，而是帮助测试人员提高效率。
例如：
自动生成测试用例，自动生成测试数据，自动生成 Mock

# 2.AI 写自动化脚本
例如：
```
帮我写 Playwright 登录测试
```

# 3. AI 自愈测试（Self-healing）
以前按钮：id=login
后来开发改成：id=loginBtn
脚本跑失败了

AI分析 DOM发现：按钮文字一样，位置一样，颜色一样。自动修改 Locator：loginBtn

# 4. AI 自动分析失败原因（Failure Analysis）
CI：1000 个 Case Fail：18 个

以前：QA 一个个看。

现在：
AI：
```
Case 12 Timeout 原因：接口响应 12s
or 元素不存在 原因：页面改版
or 数据库连接失败
```
自动分类。

# 5. AI 缺陷预测（Prediction）
利用历史数据：
```
Git 
Commit 
Bug 
代码复杂度 
PR 
作者 
模块
```
训练模型：
预测：
```
哪些模块最容易出 Bug
```

例如：
支付模块：
```
风险：95%
```
订单：
```
20%
```

然后：
重点测试支付。

# 6. AI 自动 Review
例如：
PR：
```
新增接口
```

AI：
自动检查：
```
有没有对应测试？
有没有新增接口测试？
有没有更新 Mock？
有没有更新 API Case？
```

