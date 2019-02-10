# MAC 使用技巧

1. 合上前盖 1-2 分钟后,进入休眠模式(数据存盘,内存不用供电,不用耗电), 开机需要按电源键
  
  * 开启: `sudo pmset -a hibernatemode 25`
  * 关闭: `sudo pmset -a hibernatemode 3`

2. `Mission Control` 使用技巧

  * 打开 `Mission Control`: `Control` + `Up`
  * 快速查看窗口: 将鼠标悬停在指定窗口上, 按 `Space` 键可切换
  * `Expose` 模式下切换窗口: 
    * `Control + Down`: 开启 `Expose` 模式
    * 切换应用程序: `~下的点号`
    * 反向切换应用: `tab`
    * 选取当前应用窗口: 左右方向键

3. 点击并按住键盘快捷键(`Mission Control` 或者 `Expose`), 松手后,会消失

4. 双击`Finder` 分栏条,可自动扩大当前分栏

5. 快速关机,重启,睡眠
  
  * 关机: `Control + Option + Command + 电源键`
  * 显示关机对话框: `Control + 电源键`
  * 重启: `Control + Command + 电源键`
  * 睡眠: `Option + Command + 电源键`

6. 显示文件大小

  * 显示单个文件大小: `Command + I`
  * 显示多个文件大小: 选中多个文件 `Option + Command + I`

7. 下载文件的 "文件隔离"

  * 开启: `defaults write com.apple.LaunchServices LSQuarantine -bool FALSE`
  * 关闭: `defaults delete com.apple.LaunchServices LSQuarantine`
  
  **注意:** 这些操作需要重启电脑

8. 停止生成 `.DS_store` 文件

  * 停止: `defaults write com.apple.desktopservices DSDontWriteNetworkStores
    -bool TRUE`
  * 恢复: `defaults delete com.apple.desktopservices DSDontWriteNetworkStores`

9. 将终端命令行结果输出到 GUI 程序中

  * 输出结果到默认 GUI 程序中: `ls -al | open -f`
  * 输出结果到特定的应用程序中: `ls -la | open -f -a VSCode`
  * 输出结果到剪贴板: `ls -la | pbcopy`
  * 直接输入文字到剪贴板中: 运行 `pbcopy` 后,
    再输入需要的文字,可以使用回车换行,输入完成后 `Control + D` 结束
  * 剪贴板内容输出到文件: `pbpaste > textfile.txt`
