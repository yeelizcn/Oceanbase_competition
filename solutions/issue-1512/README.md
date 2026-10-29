# 赛题二、三：重复整数 IN 列表修复

## 提测材料

| 项目 | 内容 |
| --- | --- |
| Fork | https://github.com/yeelizcn/seekdb |
| 官方 Issue | https://github.com/oceanbase/seekdb/issues/1512 |
| 官方 PR | https://github.com/oceanbase/seekdb/pull/2196 |
| PR 编号 | 2196 |
| 修复分支 | codex/fix-1512-duplicate-inlist |
| 修复提交 | a84af8b3b4f9a65842497f8935051a89a17693e6 |
| 验证基线 | 6ff9b040095ec810f4b14bf7edc022483d85e39b |

如果比赛表单要求 PR 地址，填写完整 PR 链接；如果要求编号，填写 `2196`。
本目录记录的是代码提交与本地验证结果，不代表平台已判分或 PR 已合并。

## 修复内容

原实现只按 `IN` 列表总长度判断是否转换为 VALUES TABLE。例如 `{1,2,3}`
重复 1,024 次后会触发转换，向量查询因此增加子查询过滤等操作。

修复针对满足现有转换条件的标量整数和 NULL 列表，在达到阈值前统计不同值。
不同值数量不足阈值时保留普通 IN/NOT IN 表达式。全部原始值和参数位置都保留，
预编译语句、行值、其他类型以及显式阈值 1 继续沿用原来的判断。

`fix.patch` 包含一个源码文件和一对 mysqltest 测试/期望结果文件。
期望文件中的行末空格来自数据库原始 EXPLAIN 输出，保留以便精确比对。

## 验证结果

- Ubuntu 20.04 x86_64，使用仓库固定的编译工具链。
- 原始版本与修复版本均构建成功。
- 新回归测试在原始二进制上按预期失败：执行计划仍包含多余的值表操作。
- 修复版本正式回归：3 个用例通过、0 个失败；原始报告为 `mysqltest-result.json`。
- 测试覆盖重复列表、阈值边界、强制转换、NULL/NOT IN、无符号整数上界、
  非整数和行值回退，以及参数值变化后的结果正确性。

向量复现使用 4,096 行、8 维向量和固定随机种子。
将 `[1,2,3]` 重复 1,024 次，以种子 1024 打乱，构成 3,072 项列表。

| 检查 | 原始版本 | 修复版本 |
| --- | --- | --- |
| 两种列表的命中行数 | 均为 2,458 | 均为 2,458 |
| 完整命中 ID 集合与精确 Top-K | 一致 | 一致 |
| 扩展列表中的 VALUES TABLE ACCESS | 存在 | 消失 |
| 扩展列表中的 SUBPLAN FILTER | 存在 | 消失 |
| 本次 ANN 返回结果 | 紧凑列表 10 行，扩展列表 7 行 | 两种列表均返回相同的 10 行 |

这是执行计划和结果验证，没有复测原 Issue 的 20 万行延迟基准，也没有宣称加速倍数。

## 复跑

在修复分支的 seekdb 源码根目录执行，依赖尚未安装时先执行 `python3 build.py init`：

```bash
python3 build.py build -j24 --copt=-g0 --disk_cache= seekdb
python3 tools/deploy/mysqltest.py run \
  --base-dir "$PWD/.competition-test/db" \
  --work-dir "$PWD/.competition-test/results" \
  --port 12884 \
  --test-set expr.integer_inlist_rewrite \
  --test-set expr.func_equal \
  --test-set expr.expr_nseq \
  --parameter memory_budget=8G \
  --parameter cpu_count=4 \
  --parameter vector_memory_limit=2G \
  --parameter datafile_size=2G \
  --parameter datafile_maxsize=4G \
  --parameter log_disk_size=2G
```

测试工具会创建并清理指定的测试数据库目录，运行前应确认该路径专用于测试。
