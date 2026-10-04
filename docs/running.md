# 运行测试与保存进度

## 环境和代码入口

本地打开D:\Desktop\Raft，唯一代码入口是project/src/raft1/raft.go；文档源为project/README.md和project/docs。public-notes是脚本生成的公开副本。官方Go模块名保留6.5840，测试使用WSL Ubuntu-24.04；最近验收环境为Go 1.22.2。官方测试依赖节点进程及Unix socket，先make build生成节点启动程序。

以下是已完成阶段的测试命令；不同测试用途分开，辅助测试不能替代官方验收。

## PowerShell测试命令

在 PowerShell 中执行（需要 WSL Ubuntu-24.04）：

```powershell
# 官方基础一致性
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "^TestBasicAgree3B$" -count=1 -timeout=120s'
# 官方完整3A、3B、3C回归（排除自编辅助测试）
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "^Test(InitialElection|ReElection|ManyElections|BasicAgree|RPCBytes|FollowerFailure|LeaderFailure|FailAgree|FailNoAgree|ConcurrentStarts|Rejoin|Backup|Count|Persist[123]|Figure8|UnreliableAgree|Figure8Unreliable|ReliableChurn|UnreliableChurn)3[ABC]$" -count=1 -timeout=600s'
# 自编辅助 Go 测试，不能替代官方验收
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project/src && go test -v -race ./raft1 -run "^TestGuided" -count=1 -timeout=60s'
```

## Makefile入口

在WSL的project目录中，make build只构建节点程序；make test构建并运行官方3A，不能将它的PASS当作3B或3C通过；make test-guided运行自编辅助测试。完整官方3A～3C使用上面的明确筛选命令。

旧cpp-raft和cpp-lab3属于另外的C++练习，保留在原目录，结果不计入官方Go验收。

## 同步、提交和上传

修改实现后验证相应行为，在[验收记录](verification.md)登记实际结果，再用明确描述本次改动的提交信息保存。仅改文档时检查内容、链接与同步，不重复运行协议测试。

```powershell
cd D:\Desktop\Raft
# 只生成公开文档副本，不提交
.\保存进度.ps1 -SyncOnly
# 同步文档并提交到两个本地仓库
.\保存进度.ps1 -Message "本次实际修改的内容"
# 同步、提交并上传两个仓库
.\保存进度.ps1 -Message "本次实际修改的内容" -Push
```

脚本不运行测试。完整代码上传到私有raft-study-private，公开raft-study只放文档；访问与发布细节见[GitHub说明](github.md)。旧MIT目录和C++练习没有删除，日常实现以project为准。
