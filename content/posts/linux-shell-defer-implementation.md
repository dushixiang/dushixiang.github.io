+++
date = '2023-08-02T00:00:00+08:00'
draft = false
title = 'Linux Shell 实现类似 golang defer 的功能'
+++

### 基本信息

trap命令用于指定在接收到信号后将要采取的动作。trap命令有两种格式。

1. trap [COMMANDS] [SIGNALS] 此格式用来指定在接收到 SIGNALS 信号列表后将执行的命令。 示例：

    ```shell
    trap "echo 'Ctrl+C is trapped.'" SIGINT
    ```
    输出将会在你按下 Ctrl+C 时显示，即在接收到 SIGINT 信号时。

2. trap - [SIGNALS] 此格式用来重置SIGNALS的默认处理程序。

    ```shell
    trap - SIGINT
    ```
    该命令将恢复 SIGINT 信号的默认处理程序。 一旦指定了信号的处理程序，那么在脚本执行过程中，都会保持指定的动作，直到脚本执行结束。如果你想在脚本执行完毕后，不再处理这个信号，你可以使用trap - SIGNALS来重置信号的默认处理程序。


你也可以使用 trap -l 列出所有的信号，或者 trap 命令 获取当前的 trap 设置。

以下是一些常见的信号

| 信号值 | 信号名     | 描述                                                |
| --- | ------- | ------------------------------------------------- |
| 1   | SIGHUP  | 终止控制进程或者进程组                                       |
| 2   | SIGINT  | 通常由于终端的挂断或者死亡而导致进程也中断                             |
| 3   | SIGQUIT | Quit信号，和SIGINT类似，但由Quit键发出，通常用ctrl+\，执行后会生成core文件 |
| 9   | SIGKILL | Kill信号，用于结束最不能终止的进程，信号不能被进程捕捉                     |
| 15  | SIGTERM | 请求中止进程，kill命令缺省产生这个信号                             |


### 示例代码

```shell
#!/bin/bash

cleanup() {
    echo "Caught Interrupt, cleaning up..."
    # 在这里放置你的清理代码。例如，删除一些临时文件。
}

exit_handler() {
    echo "Caught Terminate, exit..."
    exit 1
}

trap 'cleanup' SIGINT
trap 'exit_handler' SIGTERM

while true; do
    sleep 1
    echo "Sleeping..."
done
```

如何测试这段代码？

1. 当你按下 Ctrl+C（SIGINT）时，你会看到 "Caught Interrupt, cleaning up..." 这个信息然后脚本会继续运行。

2. 当你发送 SIGTERM 消息到这个脚本时，（比如通过 kill 命令：kill -s SIGTERM [pid] ），你会看到 "Caught Terminate, exit..." 这个信息然后脚本会退出。

### 总结

根据这个特性，我们就可以实现一些类似 golang defer 的功能，例如删除一些临时文件，关闭某个连接等等。