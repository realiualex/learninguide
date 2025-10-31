neovim 是基于 vim fork 过来的，支持丰富的插件，合理的配置，能实现与vscode IDE等类似的效果

* 安装 neovim
```
brew install neovim
```

之后输入 nvim 命令替代 vim 命令，nvim命令后面也可以跟文件夹路径，如当前路径可以直接输入 .

* 安装 LazyVim 插件，git clone后退出重新进入 nvim 即可开始安装
```
git clone https://github.com/LazyVim/starter ~/.config/nvim
```

* 安装语法提示
	* 在neovim中，输入 :LazyExtras，即可看到支持的plugins，光标上下选择到需要安装的插件，按 x 即可选中进行安装。也可以按 / 进行搜索。之后直接按 :q 退出即可。
## 初探
nvim里，上下左右的操作，就是 vim 的操作，即 H左，L右，J下，K上。  
如果要从文件编辑框，切换到左侧的neo tree目录，则CTRL + h  
在目录界面，按 l (小写L)，即光标往右移动，就可以打开文件，并进入编辑模式。同样 CTRL + h回到左侧目录  
从目录界面，如果不想打开文件，想直接进入到右侧的文件 buffer界面，按 CTRL + w 对窗口进行操作，然后按 l (小写L)  
如果是一个文件夹，多次按 l (小写L)即可打开和折叠文件夹，或者按h 进行折叠。  
在文件或文件夹上，按  r ，即可对文件进行重命名  
如果右侧打开了多个文件，SHIFT + l (小写l) 或 SHIFT + h 即可左右切换打开文件的窗口(这个打开的窗口叫 buffer)。  
如果想要关闭某一个打开的文件，按 空格 + b + d 即可关闭  
如果想要对文件自动换行，:set wrap。取消换行 :set nowrap  

## vim 技巧
### 复制
在vim里如果要选中复制，按小写的 `v` 可以按光标移动，然后按 `y` 就是复制。如果按大写 `V` 则可以一次选中一行，同样按 `y` 进行复制。如果想要选中多列编辑，那么是 `ctrl + v` ，然后按 `shift + i` 就可以一次操作多行编辑.

### 翻页
向后翻页(看以前的东西 ) `ctrl + b`，向前翻页(看新的东西 ) `ctrl + f`
