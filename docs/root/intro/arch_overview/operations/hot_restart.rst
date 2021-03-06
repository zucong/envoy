.. _arch_overview_hot_restart:

热重启
===========

易操作性是 Envoy 的主要目标之一。除了强大的数据统计和本地管理界面外，Envoy 还有 “热” 或者 “现场” 重启自己的能力。
这意味着 Envoy 可以完全重新加载自己（包括代码和配置）而不丢失任何连接。热重启功能有以下通用体系结构：

* 统计数据和一些锁要保持在共享内存区。这意味着计量仪在进程重启的时候将保持一致。
* 这两个存活的进程用简单的 RPC 协议通过 unix 域套接字进行通信。
* 新进程在从老进程进行拷贝监听套接字之前，需要完成自己的初始化（如加载配置，执行初始化服务发现和健康检查）。然后新进程开始监听，
  并且通知老进程开始消退（draining）。
* 在消退（draining）阶段，老进程会尝试优雅的关闭已经已有链接。具体如何关闭这取决于过滤器的配置。消退（draining）时间可以
  通过:option:`--drain-time-s` 来配置，并且随着时间的推移，消退（draining）会变得更迅速。
* 在老进程消退（draining）之后，Envoy 的新进程通知老进程关闭自己。这个时间可以通过:option:`--parent-shutdown-time-s` 选项来配置。
* Envoy 的热重启设计使得即便 Envoy 的新进程和老进程运行在不同的容器内也能正确的工作。新老进程之间的通信只使用 unix 域套接字。
* 用 Python 写的一个重启器/父进程示例包含在了 Envoy 的源码中。这个父进程可以用在标准的进程控制工具程序，比如 monit/runit 等。

Envoy 的默认命令行选项假定它在给定的主机上只启动运行一个 Envoy 进程：一个活跃的 Envoy 服务进程，可能还有一个在消退（draining）
的 Envoy 服务进程，该进程会像上面所讲的最后退出。:option:`--base-id` 或 :option:`--use-dynamic-base-id` 选项可能
用于允许多个配置明确的 Envoy 在同一个主机上运行，并且可以独立的热重启。
