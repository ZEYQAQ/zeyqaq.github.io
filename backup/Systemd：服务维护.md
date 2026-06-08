systemd将一切系统资源（服务、挂载点、设备、定时器）视为 Unit。比如最常用的**`.service`**: 是后台运行的程序，**`.timer`**: 定时任务，替代传统的 cron。**`.mount`**: 文件系统挂载。**`.target`**: 一组单元的集合（类似于运行级别，如 `multi-user.target` 表示多用户命令行环境）。

管理这些Unit需要用systemctl，日常最常用的是

| 操作                | 命令                                | 说明                                                         |
| ------------------- | ----------------------------------- | ------------------------------------------------------------ |
| 启动服务            | systemctl start <name>              | 立即启动                                                     |
| 停止服务            | systemctl stop <name>               | 立即停止                                                     |
| 重启服务            | systemctl restart <name>            | 重启服务进程                                                 |
| 重载systemd本身配置 | systemctl daemon-reload             | 当你修改、添加或删除了 .service 单元文件内容时。不重启，平滑加载配置。 |
| 重载应用程序配置    | systemctl reload <name>             | 当你修改了应用程序本身的配置文件（如 .conf 或 .yaml）时。不重启进程，重新加载配置文件 |
| 开机自启            | systemctl enable <name>             | 创建软链接，开机启动                                         |
| 禁止开机启动        | systemctl disable <name>            | 取消开机启动                                                 |
| 查看状态            | systemctl status <name>             | 查看运行状态、最后几行日志                                   |
| 列出服务            | systemctl list-units --type=service | 查看系统中当前所有的服务单元                                 |

排查问题需要用到journalctl，日常用到的是

| 功能类别           | 命令                                      | 说明                                        |
| ------------------ | ----------------------------------------- | ------------------------------------------- |
| 查询所有日志       | journalctl                                | 查看系统收集的所有日志记录                  |
| 查询特定服务日志   | journalctl -u <name>                      | 仅筛选显示指定服务的相关日志                |
| 实时追踪日志       | journalctl -u <name> -f                   | 实时滚动查看服务最新输出（类似 tail -f）    |
| 查询最近 X 条日志  | journalctl -n <数量>                      | 查看指定服务最近的 N 条日志记录（如 -n 50） |
| 查询最近 1小时日志 | journalctl -u <name> --since "1 hour ago" | 查看过去一小时内的日志记录                  |

.service文件是systemd的维护目标，放在 `/etc/systemd/system/` 目录下。

文件内容一般为：

```
[Unit]
Description=My Custom Application
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/my-app
ExecStart=/usr/bin/python3 /opt/my-app/main.py
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

编写完成后必须执行的步骤：重新加载配置： `systemctl daemon-reload`并且启动并设置自启： `systemctl enable --now my-app.service`

其中，`[Unit]` 段落的核心作用是描述该服务是什么，以及它与其他服务之间的依赖和启动顺序关系。

| 参数          | 说明                                                         | 示例                                      |
| ------------- | ------------------------------------------------------------ | ----------------------------------------- |
| Description   | 对服务的简短描述，用 systemctl status 时会显示。             | Description=Prometheus Blackbox Exporter  |
| Documentation | 指向文档的链接或手册页。                                     | Documentation=https://prometheus.io/docs/ |
| After         | 核心参数：定义启动顺序。该服务会在列表中的项启动之后才启动。 | After=network.target                      |
| Before        | 定义启动顺序：该服务会在列表中的项启动之前启动。             | Before=multi-user.target                  |
| Requires      | 强依赖：若列出的单元启动失败，本服务也会启动失败。           | Requires=network-online.target            |
| Wants         | 弱依赖：若列出的单元未启动，不影响本服务启动，仅尝试启动它们。 | Wants=syslog.target                       |

其中，`.target` 理解为“系统的运行状态点”**或**“逻辑分组”。在传统的 SysVinit 系统中，这对应于“运行级别（Runlevel）”。`systemd` 将这些级别细化为了 `.target`。 `target` 文件本身就是一个特殊的 Unit 文件，它的作用是**将一组相关的服务“打包”在一起**。当系统启动时，它不会一个一个去启动成百上千个服务，而是通过“启动某个 `target`”来批量拉起该状态下必须的服务。

常见的target：

| 目标 (Target)         | 核心状态描述                       | 典型应用场景                                           |
| --------------------- | ---------------------------------- | ------------------------------------------------------ |
| network.target        | 网络协议栈已就绪（网卡驱动加载）。 | 绝大多数基础网络服务的起点。                           |
| network-online.target | 网络已连接并成功获取 IP 地址。     | 需要联网才能工作的程序（如 NTP、云同步、数据库连接）。 |
| multi-user.target     | 多用户命令行环境，系统准备就绪。   | 绝大多数后台服务（守护进程）的默认启动目标。           |
| graphical.target      | 包含图形界面（GUI）的环境。        | 桌面操作系统（如 Ubuntu Desktop）的最终启动目标。      |

其中，`[Service]` 段：定义“如何运行”。这是服务配置的核心，决定了程序的启动命令、运行身份和重启策略。

| 参数             | 说明                                                         | 示例                                   |
| ---------------- | ------------------------------------------------------------ | -------------------------------------- |
| Type             | 定义进程启动类型（simple 最常用，表示主进程就是 ExecStart）。 | Type=simple                            |
| ExecStart        | 最关键：启动程序的完整路径及参数。                           | ExecStart=/usr/bin/python3 /opt/app.py |
| WorkingDirectory | 指定程序运行的工作目录。                                     | WorkingDirectory=/opt/app              |
| User / Group     | 以哪个用户/用户组的身份运行（安全建议：尽量不用 root）。     | User=webuser                           |
| Restart          | 重启策略（on-failure 表示只有崩溃时才重启）。                | Restart=on-failure                     |
| RestartSec       | 重启前的等待时间。                                           | RestartSec=5s                          |
| Environment      | 设置环境变量（如 API 密钥、端口）。                          | Environment="PORT=8080"                |

其中，`[Install]` 段：定义“安装行为”。这一段配置决定了当你执行 `systemctl enable` 时，systemd 会做些什么（即建立哪些软链接）。

| 参数       | 说明                                                  | 示例                       |
| ---------- | ----------------------------------------------------- | -------------------------- |
| WantedBy   | 指定该服务归属哪个 Target。通常填 multi-user.target。 | WantedBy=multi-user.target |
| RequiredBy | 与 WantedBy 类似，但表示强依赖关系（较少用）。        | RequiredBy=network.target  |
| Alias      | 设置服务的别名，方便通过不同名字启动。                | Alias=my-service.service   |

---

深度功能：

1、资源控制

`systemd` 集成了 `cgroups`，这意味着可以直接在 `.service` 文件中限制程序的资源使用，防止某个进程“吃光”系统资源。

**限制 CPU 使用率：** `CPUQuota=50%` (限制该服务最多使用 50% 的单核 CPU)。

**限制内存占用：** `MemoryMax=512M` (达到此限额时，内核可能会杀掉进程或使其分配受阻)。

**限制最大进程数：** `TasksMax=10` (防止 fork 炸弹)。

2、自动化重启策略 

除了简单的 `Restart=always`，还可以配置更细致的恢复逻辑：

**`StartLimitIntervalSec=0`**：默认情况下，如果服务在 10 秒内重启超过 5 次，systemd 会放弃重启。

**`Restart=on-failure` vs `Restart=always`**：`on-failure` 只有在非正常退出（状态码非 0）时重启；`always` 只要进程退出就重启（即使是通过 `systemctl stop` 停止的，也会被自动拉起）。

3、环境变量与配置文件分离

```
[Service]
# 在 /etc/default/my-app 中以 KEY=VALUE 格式定义变量，把敏感信息存进去，避免密码明文暴露在service
EnvironmentFile=/etc/default/my-app
ExecStart=/usr/bin/my-app --api-key=$API_KEY
```

4、类似于corntab的定时任务

一个 `.timer` 文件对应一个 `.service` 文件。可以设置 `OnBootSec=15min`（开机后 15 分钟执行），或者 `OnUnitActiveSec=1h`（上次运行结束后 1 小时再次执行，比 cron 的固定时间更灵活）。

5、服务类型进阶

 `Type=simple`：标准

**`Type=forking`**：适用于后台守护进程（Daemon），程序启动后会自己 fork 出子进程然后父进程退出。

**`Type=notify`**：适用于与 systemd 有深度交互的程序，程序启动完成后会主动通知 systemd“我已经准备好了”。

6、排查问题

查看服务启动耗时排行： `systemd-analyze blame`

以图形化方式查看启动链： `systemd-analyze dot | dot -Tsvg > system.svg`

实时监控服务的资源消耗： `systemd-cgtop`（界面类似 `top` 命令，但按 cgroups 分组）。

查看当前所有挂载点及其依赖： `systemctl list-units --type=mount`