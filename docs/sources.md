# 参考来源与贡献边界

- MIT 官方骨架：git://g.csail.mit.edu/6.5840-golabs-2026
- [Lab 3 说明](https://pdos.csail.mit.edu/6.824/labs/lab-raft1.html)
- [Raft 原论文](https://raft.github.io/raft.pdf)：主要参考 Figure 2、5.2 选举、5.3 日志复制、5.4 安全性与旧任期提交限制。
- [MIT 协作政策](https://pdos.csail.mit.edu/6.824/labs/collab.html)：不公开实验答案，完整代码保存在本地或私有仓库。

本文整理自己的学习代码与交流记录。主要练习状态转换、投票、日志复制、冲突修复、多数派提交、顺序交付、持久化恢复及冲突回退优化，Go 写法、接口和部分发送辅助代码得到 Codex 协助。辅助单元测试由 Codex 编写；官方测试及运行框架归属官方来源。

公开版不包含完整或部分实验解答源码，不用于替代符合课程协作要求的独立提交。
