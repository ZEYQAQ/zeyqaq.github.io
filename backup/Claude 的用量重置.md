Claude 的用量限制并不是在每天午夜重置，而是采用滚动式的5小时窗口——从你发送第一条消息时开始计时。

使用crontab定时任务，可以实现每天定时调用Claude Code 给 Claude 发一条消息来锁定窗口起点。（六点至十一点，十一点至下午四点，下午四点至晚上九点）这样可以避免很多明明额度够用，但是usage爆掉的尴尬情况。


我的Macmini日常不关机，所以选用了此方案。
注，Anthropic最近更新了规则，本方案仅重置claude.ai、Claude Code 和 Claude Desktop 的每日额度。API额度已经和Plan分离开，原通过Python脚本重置API额度的不再赘述；我还没有验证crontab在Windows机器上的可行性，所以Windows用户可以关注后续回复。



```
#MacOS/Linux Terminal中运行：


# 编辑 crontab
crontab -e
# 每天早上 6:00 自动发一条消息
0 6 * * * echo "hi" | claude --print 2>/dev/null
```



<img width="580" height="329" alt="Image" src="https://github.com/user-attachments/assets/c4d0ba5d-a869-4c2c-b0c2-66162046adf2" />

Hi! How can I help you today?