# GitHub 发布与面试官访问

本地统一位于 D:\Desktop\Raft：public-notes 是公开文档版；project 是完整代码版。它们分别对应 raft-study 和 raft-study-private 两个 GitHub 仓库。二者都有自己的 Git 历史，原 MIT 仓库未改变。

## 发布步骤

两个远程仓库已创建：公开说明为 [raft-study](https://github.com/2258009564/raft-study)，完整代码为私有 [raft-study-private](https://github.com/2258009564/raft-study-private)。不要重复创建；只编辑 project，日常更新使用：

```powershell
cd D:\Desktop\Raft
.\保存进度.ps1 -Message "说明实际完成内容与测试结果" -Push
```

若已有同名远程仓库，先检查该仓库的内容和可见性，不要覆盖或强制推送。

## 简历怎么放

简历项目链接使用公开 `raft-study` 仓库。现阶段可以说明“基于 MIT 官方骨架，在分步指导下实现 Raft 选举、日志复制、冲突修复、多数派提交与顺序交付，通过 MIT 6.5840 2026 Lab 3A、3B 官方测试与竞争检测”；3C、3D 未实现，不能写成完整 Raft、生产级系统或分布式 KV。实际贡献和结果见 README 与测试记录。

README 区分学习者实现、官方框架和辅助工作。完整源码需要时可以通过私有仓库审核，不必把实验答案公开。

## 面试官怎么访问源码

拿到对方 GitHub 用户名后，在私有仓库的 Settings → Collaborators 中邀请指定用户，对方接受邀请后可访问源码。简历链接本身仍可公开访问。不要为了方便生成公开下载链接或将源码放进公开 Release、附件、Issue 或历史提交。

公开版可以继续补充故障现象、测试结论、设计取舍和复习解释，由脚本从 project 同步；不能用计划代替验收结果。脚本不运行测试，也不审查 Markdown 正文中的敏感信息，不粘贴完整实验解答。两次上传不是原子操作，失败时保留本地提交，确认各仓库状态后重试。
