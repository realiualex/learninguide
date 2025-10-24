# tmux 和 screen

## tmux

### Session 管理

* 创建一个名字为 dev 的session` tmux new -s dev `
* 将 session 取消挂载: 在session里，先按 `CTRL +B` ，然后再按 `D`
* 查看当前有多少个session  `tmux ls `
* 连接到session ``` tmux attach-session ```。如果想连接到指定session，则是  `tmux att -t dev `
* 删除某个session ` tmux kill-session -t dev 
* 在已经连接的session 中，可以按 `CTRL +B`，然后再按 `$`，这样就能直接重命名session
* 在某个session中，如果想要直接切换到另外一个session，先按` CTRL +B`，再按 w，就可以选择session

### 窗口管理 与pane

### pane 的操作

* 上下拆分：在 tmux 的session里，按`CTRL +B`，然后按双引号 `"`
* 水平拆分: 先按`CTRL +B`，然后按百分号 `%`
* 切换pane: 先按`CTRL +B`，然后按上卡左右键
* 调整pane大小：先按 `CTRL +B`，然后按 `OPTION`+上下左右键(windows键盘是ALT+上下左右)
* 查看 pane 的编号:` CTRL+B`，再按` q`
* 快速切换pane：先按`CTRL+B`查看编号，然后直接输入编号，就可跳转到编号pane里。如果想要两个窗格来回切，则是 `CTRL +B `在按分号`;`
* 将pane设置为全屏：`CTRL +B`再按` z `就将pane 切换为全屏。再按一遍就切换回来
* 将pane设置为window：在某个pane里，按`CTRL +B` ，在按感叹号` ! `就会将当前的pane转换为window
* 将window挪回到pane里：CTRL +B 然后按冒号进入命令模式: ```join-pane -s 1 -t :0```，这个指的是，将窗格pane移动到0号window的0号pane位置
* 窗格pane位置调整： 按 CTRL +B + 空格来回进行切换，就能自动切换。如果想要手动调整，`CTRL +B`，再按`{ `进行逆时针旋转(窗格 3，2，1，0)，如果 CTRL +B 再按` } `则顺时针 (0,1,2,3) 的顺序进行切换
* 在窗格里同时执行相同操作：按`CTRL +B`，然后按冒号`: `进入命令模式，输入 ```setw synchronize-panes``` 回车，就能同步所有窗格执行相同操作。再执行相同的一遍，就取消同步了.

### window的操作

* 在tmux会话里，按 `CTRL +B`，然后按`C`就可以创建一个window。
* 切换window：`CTRL+B`，然后按数字`0`，`1`，`2`就可以切换指定window。也可以按 `CTRL+B`，再按` n` 按顺序切换window。按 `Ctrl+B`，再按`p`进行倒序切换。最近两个来回切换: `CTRL+B` 在按`L`
* 重命名window：在当前window中，输入`CTRL+B`，再按`:` 进入命令模式，输入``` rename-window window_name ``` 
* 

* 换window和session：`CTRL +B `再按 `W`就可以选择window，pane或者session

## session 内操作

* 滚屏：在tmux session内，如果要向上下滚动屏幕，先按 `CTRL +B`，然后按 `[`，之后按上下左右键盘就能移动光标了。默认情况 tmux 窗口内是没法通过鼠标上下滚动的，也可以修改 ~/.tmux.conf 文件，加上 ``` set -g mouse on ``` 就能使用鼠标(或mac的触摸板)进行滚屏

* 复制：如果启用了鼠标滚屏，那么想要复制的话，需要按住 OPTION(windows ALT) 按键，然后按COMMAND (windows CTRL) +C，进行复制 。如果想用键盘操作直接拷贝，则先按 CTRL +B，然后按 \[，就进入了复制模式。之后可以移动光标进行选择，选到要拷贝的位置，按`CTRL + 空格`，开始移动光标进行拷贝，到结束位置，按 `CTRL+W`，结束拷贝。然后按 `CTRL +B`，再按 `]` 进行粘贴。拷贝的内容实际在buffer中存储，如果想要看下buffer中的内容，可以按 `CTRL +B`，在按冒号` : `进入命令模式，输入 `show-buffer `就能看到

* 在复制模式中 (即 `CTRL +B之后按 [ `)，可以通过 `CTRL + 上下按键`，一次滚动半屏。可以按 `CTRL + s`. 进行搜索，在搜索中，按 `n` 进行下一个，`N`进行上一个搜索。在复制模式中，按 `CTRL +A 到行首`，`CTRL +E `到行尾。

### 复制模式改成 vi

在 `ctrl_b + [` 的选择模式下，用的是 emac 兼容的，但我们可以改成 vi 兼容命令。这样方便搜索。需要改 `~/.tmux.conf` 文件，加上这一行
`set -g mode-keys vi`

之后在编辑模式下，复制的时候就按 空格+回车，而不是` ctrl + 空格`，然后 `ctrl + w` 里

  

## 命令模式

在tmux里，按 CTRL +B，然后按 : 就能进入命令模式，比如命令模式下可以对窗口进行管理

```
#垂直切割
split-window -h
# 水平切割
split-window -v
```

 



## 插件管理 （tpm）

使用[tpm](https://github.com/tmux-plugins/tpm) 可以管理插件，改完配置后执行 ` tmux source-file ~/.tmux.conf` 就可以生效

之后可以安装其他插件，比如 [dracula tmux](https://draculatheme.com/tmux)

```shell
set-option -g default-shell /bin/zsh
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-prefix-highlight'
set -g @plugin 'tmux-plugins/tmux-sidebar'
set -g @plugin 'dracula/tmux'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'


set -g @dracula-plugins "git cpu-usage ram-usage time"
#set -g @dracula-refresh-rate 5
set -g @dracula-network-bandwidth en0
set -g @dracula-network-bandwidth-interval 0
set -g @dracula-network-bandwidth-show-interface true

set -g @continuum-save-interval '15'
set -g @continuum-restore 'on'
set -g @resurrect-capture-pane-contents 'on'

run -b '~/.tmux/plugins/tpm/tpm'

#set -g mouse on
```


在准备好 ~/.tmux.conf 文件后，先source一下 ``` tmux source ~/.tmux.conf ```。之后进入到 tmux的窗口中，然后按 CTRL+B，之后按 I (大写i)，就会安装插件

装好上述插件，按 CTRL +B ,然后按 TAB 就可以列出来当前路径的文件

### 主题

* 首先安装Powerline fonts[字体](https://github.com/powerline/fonts)
* 在 Item2 中，选择Profile --> Text --> Font中选择 Noto Mono for Powerline
* 之后可以安装其他主题，比如 dracula tmux，或者 nordtheme/tmux

### 会话保存

tmux 默认情况下，一旦电脑重启，会话就丢失了。

我们可以通过插件来实现将会话保存到磁盘上。这个可以通过 tmux-resurrect 这个插件实现，可以按 CTRL + b 后按。CTRL + s 进行保存(在保存的时候，tmux状态栏会有提示 saving)，开机后，输入tmux进入tmux窗口里，按 CTRL b后按 CTRL + r  进行恢复，然后 CTRL B +w 或者 CTRL B + s 进行选择session

如果想自动保存会话，则可以通过 tmux-continuum 这个插件实现，这个插件依赖于 tmux-resurrect





## Screen

* 直接输入 screen 就可以开启一个会话

* detach: 在会话里，按CTRL +A，再按D取消挂载
* 查看会话: screen -ls
* 恢复会话: screen -r 会话ID
* 