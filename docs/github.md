# GitHub 发布与面试官访问

本地统一位于 D:\Desktop\Raft：public-notes 是公开文档版；project 是完整代码版。它们分别对应 raft-study 和 raft-study-private 两个 GitHub 仓库。二者都有自己的 Git 历史，原 MIT 仓库未改变。

## 发布步骤

GitHub CLI 当前未登录，无法自动创建远程仓库。完成账号登录后，在 PowerShell 运行：

```powershell
gh auth login
gh repo create raft-study --public --source D:\Desktop\Raft\public-notes --remote origin --push
gh repo create raft-study-private --private --source D:\Desktop\Raft\project --remote origin --push
```

若已有同名远程仓库，先检查该仓库的内容和可见性，不要覆盖或强制推送。

## 简历怎么放

简历项目链接使用公开 `raft-study` 仓库。现阶段可以说明“实现 Raft 领导者选举、投票与心跳，通过 MIT 6.5840 2026 Lab 3A 官方测试和针对性故障测试”；不能写成已完成完整 Raft、日志复制或分布式 KV。

README 区分学习者实现、官方框架和辅助工作。完整源码需要时可以通过私有仓库审核，不必把实验答案公开。

## 面试官怎么访问源码

拿到对方 GitHub 用户名后，在私有仓库的 Settings → Collaborators 中邀请指定用户，对方接受邀请后可访问源码。简历链接本身仍可公开访问。不要为了方便生成公开下载链接或将源码放进公开 Release、附件、Issue 或历史提交。

公开版可以继续补充自己整理的故障现象、测试结论、设计取舍和复习解释。3B 完成后再更新范围，不能用计划代替验收结果。
