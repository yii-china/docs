# 013 — 代码审查

代码审查对项目的成功至关重要，与贡献代码同样重要。

审查通过 GitHub 拉取请求处理。当请求准备好进行审查时，会添加"status: code review"标签。

[需要审查的拉取请求完整列表](https://github.com/search?q=org%3Ayiisoft+label%3A"status%3Acode+review"&state=open&type=Issues)可在 GitHub 上找到。

## 指南

- 检出拉取请求分支，在 IDE 中打开项目，了解全局。
- 拉取请求整体上是否有意义？
- 使用拉取请求中的代码运行测试和/或应用程序。这些是否正常？
- 阅读拉取请求中的所有代码行。
- 能否做得更简单？
- 是否存在安全问题？
- 是否存在性能问题？
- 留下评论时要礼貌。
- 避免代码格式化评论。

## 必需部分

- [ ] 没有代码会失败但有代码会通过的测试。
- [ ] 类型提示。
- [ ] `declare(strict_types=1)`。
- [ ] 文档：phpdoc、yiisoft/docs。
- [ ] 更新日志条目（如果包是稳定的）。
