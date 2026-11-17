# Parser 只读 Dry-run 结果｜2026-11-17

**时间：** 约 2026-11-17 07:40 UTC / 15:40 Asia/Singapore  
**来源：** 用户在 POC Eventhouse KQL Queryset 运行后提供的 3 张截图，以下数据由截图转录，**非完整 CSV 导出**。  
**运行变更：** 仅 SELECT / Query，**未执行 Function DDL、未更改 Update Policy**。

| 解析类型 | Dry-run Raw Rows | 经 Candidate 读取后非空的字段 |
| --- | ---: | --- |
| QueryExecutionAggregationReport | 77 | Query Duration 的 Average/Max/Min：**各 77**；Data Processing Duration 的 Average/Max/Min：**各 56** |
| QueryStreamingReport | 113 | SpoolingDiskReadingDuration、StreamingDuration：**各 104** |
| MashupEvaluationReport | 91 | AverageCommit、AverageWorkingSet、MaxCommit、MaxWorkingSet、TotalProcessorTime：**各 91** |

**已经证明：** 三个只读 KQL 候选可以执行，Raw JSON 的相关字段能够读出预期非空值，与字段名不匹配的根因判断一致。

**尚未证明：** Candidate 与目标表列名/顺序/类型相同、所有原始数值与单位相等、`DataSource` 转换正确、`TotalProcessorTime(ms)` 的 timespan 正确、事务性 Update Policy 接受新函数、历史 Parsed null 能回填、文件传输无丢失/重复。

**后续应保存的证据：** 3 个 Schema Check 的全列 `IsMatch` 结果；几组原始 Raw 值和 Candidate 样本值；必要时的 POC 部署前后新记录数与异常日志。

[相关测试脚本](../../kql/poc-parser-fix-20261117/README.md) · [GW-001](../../tracking/ISSUES.md) · [STATUS](../../STATUS.md)。
