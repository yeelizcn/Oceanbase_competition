# OceanBase_competition

OceanBase competition project.

## 赛题二与赛题三

- 官方仓库 Fork：https://github.com/yeelizcn/seekdb
- 修复问题：https://github.com/oceanbase/seekdb/issues/1512
- 已提交 PR：https://github.com/oceanbase/seekdb/pull/2196
- PR 编号：`2196`

修复重复整数 `IN` 列表不必要地转换为值表、影响向量检索执行计划的问题。
Linux 源码构建成功，三个回归用例通过，并完成修复前后向量查询对比。
PR 已提交；合并状态与比赛计分由官方审核和平台判定。

验证详情及可应用补丁见 [solutions/issue-1512](solutions/issue-1512/README.md)。
