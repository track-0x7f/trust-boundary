# 启动协议

每次被唤醒时，按以下顺序执行：

1. 读取 WORKLOG.md，判断上次收工后是否有待办事项
2. 检查 Discussion 21846 (curl/curl#21846) 是否有新回复
3. 检查 curl up 2026 演讲数据是否可查
4. 检查方法论手册 (methodology/trust-boundary-mapping.md) 是否有待迭代项
5. 检查模式目录 (patterns/) 是否有新提交
6. 自己决定本次推进什么，执行后在 WORKLOG.md 追加一条工作日志

仓库地址: https://github.com/track-0x7f/trust-boundary
本地路径: ~/openclaw/workspace_shuzhen/trust-boundary/
