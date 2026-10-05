# 3D学习准备：快照与日志压缩

准备日期：2026-10-05。历史准备稿写于2026-10-05；2026-10-06已完成3D七项官方验收，下文保留原教学路线供复习，计划措辞不是当前未完成状态。本文是教学计划与设计建议，不是已完成能力或新的进度台账；唯一续接指针仍在D:/Desktop/MIT-6.5840/learning。核心算法由学习者逐步编写，不提前填写Snapshot或InstallSnapshot函数。

## 一、这次要解决什么

3C保存任期、投票和完整日志，长期运行会让保存数据和重放成本增长。3D让服务层提供已经执行到某个位置的状态快照，Raft保留对应边界及后续日志；落后节点需要的前缀已被删除时，领导者先发送快照，再接续日志。快照字节属于服务层，Raft负责边界、保存、传输与交付，不解析业务内容。

规则参考[MIT 2026 Lab 3D](https://pdos.csail.mit.edu/6.824/labs/lab-raft1.html)与[Raft论文第7节、Figure 12/13](https://raft.github.io/raft.pdf)。实验使用整份快照一次发送，不实现分块。具体接口和初始化约定以本地2026代码为准，不照搬旧版本CondInstallSnapshot教程。

## 二、先走完三条时间线

### 本机压缩：谁生成快照

A的服务层已经执行到位置8，Raft提交位置为10，日志到12。服务层将“执行到8后的业务状态”编码为S8，调用A的Snapshot(8, S8)。Raft记住边界8及该条目的任期，只保留边界占位和位置9～12，连同任期、投票一起保存。9、10可能尚待交付，11、12尚未提交；都不能因这次快照被删掉。Snapshot不是领导者专用，跟随者的服务层同样可以调用。

### 远程追赶：快照不是拉票消息

A已压缩到8，末尾12；B只有位置1～3，A给B的nextIndex是4。请求所需前缀已不在A内存中，不能访问log[3]或把起点硬夹到9假装B已经接上。A发送带边界(8, term8)和S8的InstallSnapshot。B接受后保存并向自己的服务层交付快照，随后接收9～12。快照边界之后的本地日志能否保留，要核对边界位置与任期；旧快照不能令服务层倒退。快照只证明覆盖到8，不能把matchIndex直接写成12。

### 重启：协议恢复和业务恢复分开

保存容器中有S8、边界8及后缀9～12。当前server.go先取一份ReadSnapshot结果，再创建Raft；服务层从S8恢复自己的状态，Raft从保存状态恢复任期、投票、边界及日志，内部提交/交付下界从边界开始。下一条普通交付应为9，不能从1开始，也不能再次交付同一个S8。若运行期间接到更新的远程S10，应经applyCh交付S10后再交付11。恢复后日志仍有12不代表12已经提交。

## 三、压缩后的两个坐标系

建议保留log[0]作为“快照边界占位条目”，保留边界任期、清除Command引用；初始边界是0，兼容现有哨兵。新增lastIncludedIndex作为全局边界位置，lastIncludedTerm作为边界任期（或与占位Term一致，明确唯一写入约定），以及本机保留的snapshotBytes。全部进度及RPC索引继续使用全局位置；切片访问先减去边界。

| 状态 | 全局日志位置 | 本地切片下标 |
| --- | --- | --- |
| 边界占位 | 8 | 0 |
| 首条保留命令 | 9 | 1 |
| 最后一条命令 | 12 | 4 |
| 已压缩的位置 | 3 | 不可访问 |

映射是offset = index - lastIncludedIndex，末尾是lastIncludedIndex + len(log) - 1。类比ACM中的坐标平移：删掉vector前缀后，题目的原编号并不会重编号。边界8本身不再作为普通命令执行，但其任期仍用于核对位置9的前置条件。

建议先定义三个小工具：lastLogIndexLocked返回全局末尾；logOffsetLocked转换已检查范围的位置；logTermLocked查询保留范围内的任期（含边界）。名字和返回形式可以教学时确定；越界必须由调用者或工具明确拒绝，不能偷偷夹取合法下标。

## 四、函数职责与改动范围

| 函数或接口 | 3D需要解决的事 | 锁与边界 |
| --- | --- | --- |
| 新索引工具 | 集中表达末尾、偏移和边界任期 | 持锁调用；先检查范围 |
| RequestVote、startElectionLocked | 比较/发送真实全局末尾，不能把切片长度当全局位置 | 保留先任期后位置规则 |
| becomeLeaderLocked、Start | 初始化复制进度、返回追加位置均用全局编号 | 持锁，不重编号 |
| buildAppendEntriesLocked | 将全局nextIndex转换成切片后缀 | nextIndex不超过边界时转快照路径 |
| AppendEntries | 区分已压缩、边界、保留范围和末尾之外；修复后缀使用偏移 | 保留任期与提交单调性 |
| handleAppendEntriesReply | 冲突提示使用全局编号，旧回复过滤保留 | 不把本地长度作为远端全局起点 |
| advanceCommitIndexLocked、applier | 遍历全局位置，经偏移取条目；区分快照与普通消息 | 交付在锁外，并保持唯一交付序列 |
| persist、readPersist | 增加边界元数据，保存时保留当前快照 | 同一Save提交配套状态和快照 |
| Snapshot(index, snapshot) | 接收本服务已执行前缀，压缩并保存，释放旧引用 | 自行加锁；重复边界不倒退 |
| Make | 恢复边界及快照字节，在协程启动前重建下界 | 服务层自行恢复初始快照 |
| InstallSnapshotArgs/Reply | 传递任期、领导者、覆盖边界、边界任期及快照字节；回复任期 | RPC字段导出，大写首字母 |
| InstallSnapshot | 接收远端快照，检查任期与陈旧性，处理后缀，保存并安排交付 | 不持Raft锁等待applyCh |
| sendInstallSnapshot及回复处理 | 锁内准备消息、锁外RPC，有效回复确认边界进度 | 先承认更高任期，过滤旧任期结果 |
| sendHeartbeats | 按每个跟随者nextIndex选择日志或快照 | 不在锁内等网络 |

函数名是计划建议，不代表已存在。无需修改raftapi.Raft接口或官方测试；当前API没有CondInstallSnapshot，InstallSnapshot作为Raft导出RPC方法由通信框架注册。

## 五、按小阶段实施

| 阶段 | 学习者任务 | 验收与停止点 |
| --- | --- | --- |
| D1：坐标模型 | 新增边界字段及索引工具，初始边界保持0 | 人工核对边界8例子；新工具辅助检查，再回归3A～3C |
| D2：全局索引接入 | 分轮替换选举/Start/初始化，再替换复制/提交/交付 | 原行为在边界0下保持；构造非零边界检查越界 |
| D3：本地快照 | Snapshot、保存保留、恢复边界及初始快照 | 通过官方TestSnapshotBasic3D；此时远程安装可尚未实现 |
| D4：远程安装 | 定义消息，实现接收、发送分支和回复更新 | 通过可靠网络断连安装测试 |
| D5：交付并发与恢复 | 整理待交付快照、旧消息、重启和丢包场景 | 其余3D及所有旧阶段回归；失败时先给最小反例 |

这是多个学习块，通常需数个小时乃至分两次完成，具体耗时取决于首次索引改造和并发调试；不承诺一晚全部通过。首次正式讲解只进入D1的一个小任务，不同时让学习者改所有函数。

## 六、当前实现尤其要处理的陷阱

1. persist现在每次Save(bytes, nil)。进入3D后，普通任期或日志保存也必须保留已有快照，否则第一次保存后的投票就可能清空快照。状态边界与快照必须对应同一位置。
2. 当前applier锁外发送后才更新lastApplied。服务层一收到命令即可回调Snapshot，这时Raft自己的lastApplied可能还没推进；不能机械地用index > rf.lastApplied拒绝本地快照。应以服务层已应用的契约、提交边界和日志范围判断，教学时用具体并发时间线定规则。
3. InstallSnapshot不能直接在持锁RPC里等待applyCh。建议远端接收设置待交付快照，由唯一applier串行输出；需要明确待交付/已交付的区别，不让旧在途命令在快照后出现。更大的快照可以覆盖等待中的旧快照，但应用协程完成旧快照后不能误删新快照或倒退记录；若其他路径会改变lastApplied，原来的lastApplied++也需重新审视。
4. 不能只写log = log[offset:]就认为内存已释放：保留的切片可能仍引用旧底层数组；新切片和边界占位不得保留旧Command引用。测试同时检查保存体积和压缩效果。
5. 3C的Nxtbgidx、请求前置位置、Start返回值、matchIndex、commitIndex均是全局编号。仅修改Snapshot函数而不改索引使用会破坏旧链路。
6. pending快照或RPC消息中的字节需要明确复制/只读约定；状态机字节对Raft不透明，不读里面的业务位置来代替RPC边界。
7. 不把“发出快照”“远端保存”“远端服务层已安装”混成一个状态；回复用于复制进度，applyCh用于本机服务层交付。旧快照不倒退，旧请求携带更高任期仍先退位。

## 七、历史D1起点（已完成）

文件：D:/Desktop/Raft/project/src/raft1/raft.go。先讲D1坐标表及完整本机快照例子，再只在Raft字段处添加边界位置字段，并在Make显式初始化为0；暂不改Snapshot函数体。依据论文第7节保留快照最后位置与任期的要求。教学时先给中文用途注释和这几行声明/初始化语法，让学习者写；检查字段和初始化后，再给lastLogIndexLocked的小工具任务。不要用重复背诵题阻挡推进。

验收：源码仍能编译；初始log[0]、边界0与当前行为一致；非零边界的映射讲清后再进入工具函数。只有定义/初始化正确不能宣布快照完成。辅助测试可以由助手准备，但默认提供命令由学习者运行，除非获得新的明确代跑授权。

## 八、测试命令（实际结果见verification.md）

```powershell
# 索引改造完成后的旧阶段回归；TestGuided也会被选中，报告时分开
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "3A|3B|3C" -count=1 -timeout=600s'
# 本地压缩第一关
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "^TestSnapshotBasic3D$" -count=1 -timeout=120s'
# 远程安装可靠网络第一关
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "^TestSnapshotInstall3D$" -count=1 -timeout=120s'
# 全部官方3D；没有3D辅助测试时恰好7项，以后用TestSnapshot前缀明确区分
wsl -d Ubuntu-24.04 -- bash -lc 'cd /mnt/d/Desktop/Raft/project && make build && cd src/raft1 && go test -v -race -run "^TestSnapshot.*3D$" -count=1 -timeout=600s'
```

旧阶段命令只是未来验收计划，不新增当前结果。最终还需完整3A～3D回归；重复次数依据实际故障选择，不虚构通过记录。

## 九、本地官方测试核对

| 测试 | 核查目的 |
| --- | --- |
| TestSnapshotBasic3D | 本机快照和日志体积控制；此测试特意不要求远程安装 |
| TestSnapshotInstall3D | 可靠网络下断连后落后节点重连追赶 |
| TestSnapshotInstallUnreliable3D | 丢包网络下断连与快照追赶 |
| TestSnapshotInstallCrash3D | 可靠网络下落后节点崩溃、重启与追赶 |
| TestSnapshotInstallUnCrash3D | 不可靠网络下崩溃、重启与追赶 |
| TestSnapshotAllCrash3D | 全部节点重启后恢复快照及尾部，继续递增日志位置 |
| TestSnapshotInit3D | 重启后的内存快照初始化，以及后续普通保存不丢失快照 |

核查来源：src/raft1/raft_test.go:1199、1268～1353，src/raft1/server.go:49～79、128～209，src/raftapi/raftapi.go:3～34，src/tester1/persister.go:51～69。官方测试器注册NewRfsrv返回的Raft对象，因此导出InstallSnapshot方法可用Raft.InstallSnapshot路由；不需另建注册系统。测试服务约每10条命令触发一次本机Snapshot，第一次在位置9，不是等待学习者实现KV服务。

## 收尾后的实际实现

当前字段名为lstincludeidx，边界任期存于log[0].Term；只使用lastLogIndexLocked与logOffsetLocked两个索引工具。编码顺序是任期、投票、日志、边界；Save始终保留snapshotBytes。InstallSnapshot保存后设置pendingSnapshot，唯一applier优先交付快照。sendHeartbeats依据nextIndex是否越过本机边界选择日志或快照；回复只确认请求覆盖的边界。以上已实现，前文是历史任务分解，不重新执行D1。
