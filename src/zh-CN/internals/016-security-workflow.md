# 016 — 安全工作流程

安全问题通常通过[安全表单](https://www.yiiframework.com/security)发送。

如果问题直接报告到公共页面（如仓库问题或论坛主题），请获取消息并删除该问题。感谢报告者，并指向安全表单以供下次使用。

## 验证

验证问题是否有效。如果需要，请求更多信息。

## 添加安全公告

创建一个草稿 GitHub 安全公告。

### 找出严重性

1. 使用 [NVD 计算器](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator)获取 CVSS 分数。
2. 根据[评级量表](https://www.first.org/cvss/specification-document#Qualitative-Severity-Rating-Scale)选择严重性。

### 给予报告者荣誉

询问报告者是否希望因发现问题而获得荣誉。如果是，请指向他的 GitHub 帐户。

## 请求 CVE 编号

准备好后，请求 CVE。

## 准备补丁

准备一个修复问题的拉取请求。GitHub 允许在私有分支中执行此操作。

## 等待分配 CVE 编号

通常需要几天时间。

## 发布

- 在标记下一个包发布之前合并补丁拉取请求。
- 发布安全公告。
- 将 CVE 添加到 [FriendsOfPHP/security-advisories](https://github.com/FriendsOfPHP/security-advisories)。
  参见 [#488](https://github.com/FriendsOfPHP/security-advisories/pull/488) 作为示例。
