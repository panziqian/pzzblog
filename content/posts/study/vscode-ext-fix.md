---
date: 2026-03-13
title: vscode插件安装失败问题的修复
section: study
column: study
summary: vscode 更新后插件安装失败问题的修复和探索
draft: true
---

# 正文

事情的起因是笔者在上早八时打开了 vscode 并将其更新到了最新的版本，即 [1.111](https://code.visualstudio.com/updates/v1_111) 。在完成了更新之后，有几个插件显示与新版本的 vscode 不兼容，需要进行更新。 此时问题出现了，在安装插件的过程中 vscode 弹出 `安装 "Jieba" 扩展时出错。 有关更多详细信息，请查看日志。` 的报错。完整的报错如下

> [窗口] End of central directory record signature not found. Either not a zip file, or file is truncated.: Error: End of central directory record signature not found. Either not a zip file, or file is truncated.
>    at Ob (file:///Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-utility/sharedProcess/sharedProcessMain.js:450:28625)
>    at file:///Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-utility/sharedProcess/sharedProcessMain.js:450:29929
>    at /Applications/Visual Studio Code.app/Contents/Resources/app/node_modules/yauzl/index.js:40:7
>    at /Applications/Visual Studio Code.app/Contents/Resources/app/node_modules/yauzl/index.js:190:5
>    at /Applications/Visual Studio Code.app/Contents/Resources/app/node_modules/yauzl/index.js:712:5
>    at /Applications/Visual Studio Code.app/Contents/Resources/app/node_modules/yauzl/fd-slicer.js:33:7
>    at FSReqCallback.wrapper [as oncomplete] (node:fs:671:5)

起初笔者认为这是网络环境不太好导致的安装文件损坏引发的问题，于是又重新尝试安装了好几遍，但仍然无法完成安装。故笔者尝试了手动安装，即通过 vscode 下载插件文件并手动移动到 vscode 的插件文件夹下（在笔者的机器上的路径为`~\.vscode\extensions\`）。这成功的将插件装上了。但并没有解决插件无法在线安装的问题。于是笔者借助 AI 工具开始对安装的报错进行分析。

> 对 vscode 的插件存储机制进行简单的补充，vscode 的插件上是 VISX 文件，本质上就是 ZIP 文件。 从 vscode marketplace 上下载下来的插件文件的格式为 `.visxpackage` ，大多数情况下为 `.visx` 文件的打包压缩，解压即可。

报错中的 `End of central directory record signature not found` 也是笔者认为是问题在于网络环境不佳导致插件文件不完整。（事实上确实是网络环境的锅，但并不是网络质量的问题 😂 ）从报错堆栈 `at /Applications/Visual Studio Code.app/Contents/Resources/app/node_modules/yauzl/index.js:712:5` 这一行中笔者也猜想过是 vscode 所调用的解压程序 `yauzl` 的锅。但从最快解决问题的方法来说，"Retry" 思想告诉笔者重启、删除插件重新安装就能解决 99% 的问题，笔者无意深入研究和探索这个解压程序。之后笔者尝试了包括重启 vscode 、将 vscode 回退到[之前的版本](https://code.visualstudio.com/updates/v1_109)等的方法，均没有奏效。这时笔者做出了很不理智的决定——将插件文件夹及其缓存在没有备份的情况下都删除了。然而这并没有解决问题，反而让笔者无法重新找到并安装之前的插件了。（血泪教训：执行`rm -rf`的之前一定要三思，对重要文件进行备份！！这也促使笔者在之后去对配置文件进行安全的备份。）

之后在对网络的排查时却无意发现找到了问题的突破口。笔者通过 `file` 命令对下载下来的 `.visx` 文件进行类型检查时其输出为 `test.vsix: gzip compressed data, max speed, from FAT filesystem (MS-DOS, OS/2, NT), original size modulo 2^32 9404289`。 GPT 通过这条输出找到了问题的真正原因。vscode 的插件格式应该为 `Zip archive data` 而不应该是 `gzip compressed data`。因为 ZIP 文件的 EOCD(End of central directory) 结构被 gzip 包了一层，vscode 的 ZIP 解析器无法识别，最终导致了插件安装失败。GPT 也给出了被 gzip 压缩的可能原因。

笔者开启了 VPN 代理， 此时 HTTP 传输的请求头可能被改写，强制增加了 gzip 支持，导致 ZIP 文件被 gzip 再次封装。另一个可能是代理重新进行了压缩响应，使用 gzip 对 ZIP 文件进行了压缩。而 vscode 期望的是 ZIP 文件，不能正确识别 gzip 压缩的文件，故报错，造成插件安装失败。

# 备份

在这一部分中，笔者会简单列出备份 vscode 插件的命令，可能在日后会单开一篇博客介绍配置的备份。

```bash
code --list-extensions > vscode-extensions.txt
```

通过这行命令能够将 vscode 中已安装插件的列表导出至 `txt` 文件中。之后可以通过这行命令重新安装列表中的插件。

```
cat vscode-extensions.txt | xargs -n 1 code --install-extension
```



