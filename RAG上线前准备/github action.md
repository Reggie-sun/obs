

你这边要启动的不是 GitHub Actions 本身，而是**本机的 self-hosted runner**。  
我刚看了下，你的 runner 目录还在：~/actions-runner，而且现在**没有发现正在跑的 runner service**。

最直接的启动方式：

`cd ~/actions-runner ./run.sh`

启动后看到类似下面就说明好了：

`Connected to GitHub Listening for Jobs`

如果 GitHub 上已经有 queued 的 workflow，它会自动接单。

如果你不想一直占着这个终端，可以新开一个终端专门挂着，或者临时后台跑：

`cd ~/actions-runner nohup ./run.sh > runner.log 2>&1 & tail -f runner.log`

如果你想让它以后**重启后自动起来**，可以改成 service 方式：

`cd ~/actions-runner sudo ./svc.sh install sudo ./svc.sh start`

然后检查：

`sudo ./svc.sh status`

一句话建议：

- 现在先用 ~/actions-runner/run.sh 手动拉起来
- 如果你以后经常重启机器，就改成 svc.sh install + start



GitHub Actions 在你这套 retrieval/eval 体系里的作用，不是“替代本地命令”，而是**把 release gate 变成稳定、可复现、可留证据的外部执行平面**。

可以分成 4 层看。

**1. 自动执行 gate**  
本地你可以手动跑：

- make verify-retrieval-release

GitHub Actions 做的是把同一条验证链自动跑起来：

- push / pull_request 命中相关文件时自动触发
- 或手动 / 定时触发

也就是避免“本地说跑过了，但没人知道是不是同一套命令、同一套环境”。

**2. 提供统一运行环境**  
你现在已经接了 self-hosted runner。  
这意味着 GitHub Actions 会在一台固定 runner 上执行：

- runner preflight
- scripts/run_retrieval_release_verification.sh
- make verify-retrieval-release

它的意义是：

- 不靠每个人本机随缘执行
- 不靠口头说“我本地看起来没问题”
- 把 release verification 固定在一条受控运行面上

**3. 产出 artifact 和审计证据**  
这条 workflow 不只是跑一下就结束，它还会保留：

- json
- progress.log
- stdout.log
- stderr.log
- manifest.json

这些东西的价值是：

- 失败时能回看原因
- 成功时能留 release 证据
- 后续能回答“这次为什么放行”

这就是你说的“上线免责证明”里最接近证据包的那部分。

**4. 把放行结论外部化**  
如果没有 GitHub Actions，结论往往只存在于：

- 某个人本地终端
- 某个临时会话
- 某次手工执行

有了 GitHub Actions，结论会变成：

- workflow run 记录
- step summary
- manifest 里的 result_class / failure_mode / verification_summary

也就是说，“能不能上线”不再只是本地判断，而是一个外部可见、可复查的 gate 结果。

**一句话总结**  
在这个仓库里，GitHub Actions 的作用是：

**把 make verify-retrieval-release 从一个本地命令，提升成一个可自动触发、可统一执行、可保存证据、可用于放行判断的外部 release gate。**

**它不负责什么**  
它不负责：

- 替你定义样本 truth
- 替你决定 promotion
- 替你修 retrieval
- 替你解决 NOT COMPARABLE

它只负责：

- 忠实地把 gate 跑出来
- 把红绿灯和证据留住

如果你要，我下一步可以再给你一版更贴你们流程的话：  
**“本地命令 / Makefile / wrapper / GitHub Actions” 这四层分别负责什么。**