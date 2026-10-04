# 3A / 3B / 3C 验收记录

验证日期：2026-10-04（Asia/Shanghai）。验证位置：独立整理后的完整私有版；不是原目录的历史结果。

环境：WSL Ubuntu 24.04，Go 1.22.2。来源骨架提交：513cac0aef010750fc74688da8a395974a83cf7a。

## MIT 官方 Go 测试

执行 make test，先用 -race 构建官方节点启动程序，再用 -race 运行 3A。

| 官方测试 | 结果 | 输出中的耗时 |
| --- | --- | --- |
| TestInitialElection3A | PASS | 3.71 s |
| TestReElection3A | PASS | 5.99 s |
| TestManyElections3A | PASS | 7.52 s |

整体输出：PASS，ok 6.5840/raft1 18.242s。并发竞争检测未报告错误。

## 辅助 Go 测试

执行 make test-guided，11 个顶层 TestGuided 测试全部 PASS，整体输出 ok 6.5840/raft1 1.067s。使用 -race。

覆盖：更高任期退位、投票序列、选举状态转换、拉票请求生成、回复去重和旧任期隔离、心跳任期、实际 RPC、并发拉票、截止时间范围、重置事件、心跳发送。

整理时发现并修正一个辅助测试夹具问题：Make 已启动后台循环，原测试却提供多节点的空通信端点并无锁读取时间字段。修正为单节点初始化夹具并持锁读取；未修改协议行为。

## 证据边界

以上这一节是整理时的 MIT 官方 Go 3A 和自编辅助 Go 测试历史记录；当时不包含 3B。后续 3B 与联合回归证据见下节。均不包含 C++ 测试，不声明长时间压测或无限故障情形的正确性。

整理版 raft.go 的 SHA-256：78B5D6A80659DF5096409378450BF4FB13B290ABDB575D4749EB1F7685FEEEC5

该摘要对应当时整理后的 3A 源码，并非当前 3B 版本；公开仓库不附带实验解答源码。

## 3B 学习者运行结果（2026-10-04）

学习者自行运行官方 TestBasicAgree3B，开启 -race，完整输出确认 PASS，测试耗时1.45s，包总耗时2.470s。随后运行全部官方3B，并提供结尾 `PASS`、`ok 6.5840/raft1 74.452s`。全套逐项日志未在该次用户消息中提供，因此不为这一次填写各项耗时。

## 最新官方与辅助联合回归（2026-10-04）

学习者明确授权 Codex 代跑收尾回归。环境为 WSL Ubuntu-24.04，先以 -race 编译节点进程，再执行：

```powershell
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "3A|3B|^TestGuided" -count=1 -timeout=300s'
```

| MIT 官方测试 | 结果 | 测试耗时 |
| --- | --- | --- |
| TestInitialElection3A | PASS | 3.81s |
| TestReElection3A | PASS | 6.15s |
| TestManyElections3A | PASS | 7.14s |
| TestBasicAgree3B | PASS | 1.40s |
| TestRPCBytes3B | PASS | 3.04s |
| TestFollowerFailure3B | PASS | 5.33s |
| TestLeaderFailure3B | PASS | 5.90s |
| TestFailAgree3B | PASS | 5.19s |
| TestFailNoAgree3B | PASS | 4.59s |
| TestConcurrentStarts3B | PASS | 1.44s |
| TestRejoin3B | PASS | 7.19s |
| TestBackup3B | PASS | 35.53s |
| TestCount3B | PASS | 3.01s |

另外20个顶层辅助测试（旧选举/心跳11个、3B新增9个）全部PASS。整体输出 `PASS`、`ok 6.5840/raft1 90.890s`，未报告数据竞争。这里的90.890s是官方与辅助联合运行总耗时，不是单独3B耗时。

新增辅助测试覆盖：哨兵初始化、Start本地追加、投票日志新旧、接收端冲突/缺失/迟到请求、领导者初始化、请求连接点与独立复制、回复乱序与任期、多数派提交、应用顺序与通道阻塞不占锁。它们由Codex编写，不是MIT官方测试，也不构成协议形式化证明。

收尾修复了旧辅助测试手工构造节点缺少日志哨兵、领导者复制进度的问题：使夹具满足新版初始化约定，断言保留，未修改协议算法或官方测试。源码收尾只整理注释，保持非注释代码与回归前一致。该历史回归时3C、3D未运行且未实现；3A、3B各一轮回归通过，不宣称反复压力测试或生产安全认证。

## 同步工具核查（2026-10-04）

在临时克隆仓库中完成 10 项回归检查：README 和嵌套新增文档、删除文档镜像、公开误放源码时修改前拒绝、已删除但仍暂存的源码拒绝、分支异常拒绝、两仓库提交与重复执行无多余提交、错误可见性阻止上传、HTTPS/SSH 地址解析与模拟双上传、仿冒主机拒绝、其他分支历史源码拒绝。

所有检查通过。GitHub 查询和推送在测试中被模拟，没有实际网络发布；这不等于验证账号登录或真实网络上传。脚本同步 README 和 docs 下所有 Markdown，但不检查 Markdown 正文是否包含不适合公开的信息，学习文档不要粘贴完整解答。


## 3C收尾与联合回归（2026-10-05）

首次完整3C运行，8项官方测试中7项通过，TestFigure8Unreliable3C失败：`one(4485) failed to reach agreement`，包耗时182.994s。通过返回缺失日志或冲突任期段的发送起点，改进逐位置回退；分析与贡献边界见[工程案例001](engineering-cases.md)。官方测试未修改，旧辅助夹具补齐保存容器，新辅助测试覆盖保存恢复、提示编码与迟到回复。

修正后原失败场景单次PASS（45.00s），随后连续3次PASS（44.81s、39.44s、43.90s；包耗时129.169s），最新联合回归中再次PASS（46.76s）。合计5次通过，均开启-race。全部25项顶层辅助测试PASS，包耗时1.202s。

最新官方联合回归由Codex在学习者明确授权后代跑，先make build，再执行：

```powershell
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "^Test(InitialElection|ReElection|ManyElections|BasicAgree|RPCBytes|FollowerFailure|LeaderFailure|FailAgree|FailNoAgree|ConcurrentStarts|Rejoin|Backup|Count|Persist[123]|Figure8|UnreliableAgree|Figure8Unreliable|ReliableChurn|UnreliableChurn)3[ABC]$" -count=1 -timeout=600s'
```

21项官方测试全部PASS：3A共3项、3B共10项、3C共8项；包耗时254.275s，未报告数据竞争。3C各项耗时如下：

| 官方测试 | 结果 | 耗时 |
| --- | --- | --- |
| TestPersist13C | PASS | 5.99s |
| TestPersist23C | PASS | 20.64s |
| TestPersist33C | PASS | 2.98s |
| TestFigure83C | PASS | 50.12s |
| TestUnreliableAgree3C | PASS | 6.65s |
| TestFigure8Unreliable3C | PASS | 46.76s |
| TestReliableChurn3C | PASS | 17.90s |
| TestUnreliableChurn3C | PASS | 18.41s |

官方测试来自raft_test.go，自编辅助测试来自guided_*_test.go，C++练习不计入以上Go验收。随后收尾仅修改注释与文档，未再修改协议逻辑。3D尚未实现；有限次随机测试通过不能证明所有故障轨迹正确，也不作为生产系统认证。
