1、在vscode中添加插件：PlatformIO

2、

```
https://github.com/anthropics/claude-desktop-buddy.git
```

3、在vscode中打开此项目的目录。

4、使用数据线连接电脑与M5StickC Plus

5、编译与烧录，点击图中的✅与👉

<img width="1681" height="1050" alt="Image" src="https://github.com/user-attachments/assets/3a8d1ed5-8add-4a47-9519-741fcc2f98ae" />

6、在 Claude 桌面端打开 **Help → Troubleshooting → Enable Developer Mode**，打开 **Developer → Open Hardware Buddy…**，点 **Connect**，从列表里选择你的设备，打开蓝牙，输入esp32上的显示的数字（不要点击忽略此设备）

----

🤫它在睡觉：

<img width="1279" height="1706" alt="Image" src="https://github.com/user-attachments/assets/a0dc575a-bd0b-49dc-9e75-6b0642b3d894" />

---

**基本操作**

正面的 A 按钮切换屏幕页面，右侧的 B 按钮用来滚动内容。长按 A 会进入设置菜单，在菜单里可以切换宠物、调整蓝牙等。

**18 种宠物可选**

菜单里选 "next pet" 可以换宠物——有猫、企鹅、章鱼、龙、蘑菇、水豚、鸭子等 18 种，每种都有 7 套不同的动画表情。你喜欢的会自动保存。

**和 Claude 桌面端联动（重点玩法）**

配对 Claude 后才是最有趣的部分。去 Claude 桌面端开启 Developer Mode，然后打开 Hardware Buddy 连接设备。连上后：

- 没事的时候它会发呆打瞌睡
- 你开始和 Claude 对话，它会忙起来冒汗
- Claude 需要你批准某个操作时，它会闪 LED 灯提醒，你可以直接在设备上按 A 批准、B 拒绝
- 如果你 5 秒内批准了，它会开心地冒爱心
- 每消耗 5 万 token 它会升级庆祝

**体感互动**

摇一摇设备，它会晕头转向。把设备屏幕朝下放，它会进入睡眠模式恢复能量。30 秒不操作屏幕会自动关闭，按任意按钮唤醒。

**自定义 GIF 角色**

如果内置的 ASCII 宠物不够用，你还可以做自己的 GIF 角色包。项目里的 `characters/bufo/` 就是一个示例——一只小青蛙。通过 Hardware Buddy 窗口把角色文件夹拖进去就能用。

---

以下是18种预定buddy，可以提前看好再在开发板上切换：

1. Axolotl

```
}~(______)~{
}~( o  o )~{
  ( .--. )
  (_/  \_)
```

2. Blob

```
   .----.
  ( o  o )
  (      )
   `----`
```

3. Cactus

```
 n  ____  n
 | |o  o| |
 |_|    |_|
   |    |
```

4. Capybara

```
  n______n
 ( o    o )
 (   oo   )
  `------'
```

5. Cat

```
   /\_/\
  ( o  o )
  (  w   )
  (")_(")-
```

6. Chonk

```
  /\____/\
 ( o    o )
 (   ..   )
  `------'
```

7. Dragon

```
  /^\  /^\
 <  o    o >
 (   ww   )
  `-vvvv-'
```

8. Duck

```
    __
  <(o )___
   (  ._>
    `--'
```

9. Ghost

```
   .----.
  ( o  o )
  |  __  |
  ~`~``~`~
```

10. Goose

```
    (>
    ||
  _(__)_
   ^^^^
```

11. Mushroom

```
 .-o-OO-o-.
(__________)
   |o   o|
   |____|
```

12. Octopus

```
   .----.
  ( o  o )
  (______)
  /\/\/\/\
```

13. Owl

```
   /\  /\
  ((O)(O))
  (  ><  )
   `----'
```

14. Penguin

```
   .---.
  ( o>o )
 /(     )\
  `-----`
   J   L
```

15. Rabbit

```
   (\/_/)
  ( o o )
 =( v  )=
  (")_(")-
```

16. Robot

```
   .[||].
  [ o  o ]
  [ ==== ]
  `------'
```

17. Snail

```
  \\ /
   .--.
 _( oo )_
(___@@___)
 ~~~~~~~~
```

18. Turtle

```
   _,--._
  ( o  o )
 /[______]\
```

---

使用CC验证了一下它的功能，开发过程中只要选择了Bypass permission，它就不会有什么好玩的交互了，但是仍然会显示进程在屏幕下方。未来使用CC做一份汉化版的内容，今天的额度用光了，后续有空了再继续搞。<img width="1255" height="1044" alt="Image" src="https://github.com/user-attachments/assets/4dbf1eeb-c462-4838-954d-5a0bcd6448dd" />