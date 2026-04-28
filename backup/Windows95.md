github上有个怀旧项目：[[Mac电脑上运行Windows95](https://github.com/felixrieseberg/windows95)](https://github.com/felixrieseberg/windows95)

前提条件： 1、安装好 XQuartz 和 Docker。2、M系列APPLE芯片

步骤如下：

打开 XQuartz，进入菜单栏 XQuartz → Preferences（偏好设置）→ Security（安全性），勾选 Allow connections from network clients（允许网络客户端连接）。

完全退出并重启 XQuartz（Command+Q 退出，再重新打开）。

打开 终端（Terminal），运行：

```
xhost +
```

这条命令允许任何客户端连接你的 X11 显示服务器。

运行 Docker 容器：

```
docker run -it -e DISPLAY=host.docker.internal:0 toolboc/windows95
```



页面很复古，有IE浏览器和一些经典怀旧小游戏。

缺点是我发现键盘的映射关系有点错乱，尝试了不同的配置设置，也没调好。

下一步的目标可能是使用IE浏览器访问本网站。

：）

<img width="1280" height="847" alt="Image" src="https://github.com/user-attachments/assets/53e9c129-738f-4a9a-9a45-7c1443639ed2" />

<img width="1280" height="847" alt="Image" src="https://github.com/user-attachments/assets/75132689-ffa5-4e1e-8b0c-bf33e9fab851" />

<img width="1280" height="847" alt="Image" src="https://github.com/user-attachments/assets/7a598ec4-d973-478b-83b3-934551bbf774" />

<img width="1280" height="847" alt="Image" src="https://github.com/user-attachments/assets/58963043-2eab-4400-9d64-c662ecfcce3a" />