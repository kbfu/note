# 记录一次Litmus io-stress使用中发现的问题
## 背景
最近的工作中需要大量使用Litmus来生成故障来进行注入，对系统进行破坏，总算是把Litmus配置完毕后开始一步步进行workflow的开发了。

但是在进行pod-io-stress的使用中就发现很奇怪，注入根本没成功，日志也显示没什么错误，只是提示进程被杀死了，helper pod失败。

这就感觉很诡异了，想了想和以前用的时候唯一的不同是这个路径是挂载的PVC，会是这个问题吗？于是继续排查。

## 初步的判定
检查了下litmus-go这个repo中的源代码，我使用的是2.14，所以我直接`git checkout 2.14.1`这个tag了。然后找到了[这里](https://github.com/litmuschaos/litmus-go/blob/2.14.1/chaoslib/litmus/stress-chaos/helper/stress-helper.go#L124)，这就是那个helper的调用了。可以很清晰的看到他们是用了`nsutil`这个命令，然后再运行了stress-ng来进行磁盘的压力读写操作。

所以我首先排除了stress-ng命令行的错误的问题，在目标pod的容器中直接运行了stress-ng的命令和参数，确认了是可以正常运行的。

那么问题就在nsutil了，估摸着应该是一个他们自己写的用来进入容器namespace的工具，于是我就看到了他们的[nsutil repo](https://github.com/litmuschaos/test-tools/tree/master/custom/nsutil)。那么事情就简单了，直接看源代码。

## nsutil源码分析
其实main就一个简单的调用，主要内容全在[root.go](https://github.com/litmuschaos/test-tools/blob/master/custom/nsutil/cmd/root.go)中。不过一看吓一跳，他们没有mnt的参数，那怎么能对目标进程的文件系统进行操作呢？答案当然是不可能的。那我想问题应该也很简单吧，修改一下这
```go
var ns = []string{"net", "pid", "cgroup", "uts", "ipc"}
```
改成
```go
var ns = []string{"net", "pid", "cgroup", "uts", "ipc", "mnt"}
```
然后加上新的cli参数
```go
func init() {
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[0], "net", "n", false, "network namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[1], "pid", "p", false, "pid namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[2], "cgroup", "c", false, "cgroup namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[3], "uts", "u", false, "uts namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[4], "ipc", "i", false, "ipc namespace to enter")
	rootCmd.PersistentFlags().IntVarP(&t, "target", "t", 0, "target process id (required)")
	err := rootCmd.MarkPersistentFlagRequired("target")
	if err != nil {
		log.WithError(err).Fatal("Failed to mark required flag")
	}
}
```
改成
```go
func init() {
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[0], "net", "n", false, "network namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[1], "pid", "p", false, "pid namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[2], "cgroup", "c", false, "cgroup namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[3], "uts", "u", false, "uts namespace to enter")
	rootCmd.PersistentFlags().BoolVarP(&nsSelected[4], "ipc", "i", false, "ipc namespace to enter")
    rootCmd.PersistentFlags().BoolVarP(&nsSelected[5], "mnt", "m", false, "mnt namespace to enter")
	rootCmd.PersistentFlags().IntVarP(&t, "target", "t", 0, "target process id (required)")
	err := rootCmd.MarkPersistentFlagRequired("target")
	if err != nil {
		log.WithError(err).Fatal("Failed to mark required flag")
	}
}
```
最后，我对这个项目进行了编译`GOOS=linux GOARCH=amd64 go build`。之所以要这么写因为Mac肯定是没法直接调用setns函数的。

一切就绪，我将编译后的nsutil拷贝进一个测试用的pod里，参数什么的全和之前的helper pod保持一致，然后敲入命令行，成功了吗？并没有！提示我
```
invalid argument, ns-type mnt, Failed to setns
```
怎么回事？感觉检查路径，没错啊？那是哪里出了问题？经过我一番检索后，终于找到了问题，原来是Go的问题。

Go的runtime在启动后就会启动多个线程，mnt对于进程中有多线程的程序禁止enter，所以才会有这个问题，所以Go确实是没法enter mnt的，即使调用了`runtime.LockOSThread()`。
```
A process can't join a new mount namespace if it is sharing filesystem-related attributes (the attributes whose sharing is controlled by the clone(2) CLONE_FS flag) with another process.
```
至此我也总算是搞明白了为什么他们的`nsutil`没有mnt的选项，因为使用纯Go根本做不到。

## 解决办法
那么接下去就是该解决这个问题了，如果硬要用Go，那该怎么办？我想到了runc的nsenter，但是很不幸，这东西基本就是个套着Go的壳子的C程序，和直接写C基本没什么太大区别了。而且nsenter还有个问题，就是无法在目标namespace中执行没有的命令。

## nsexec
然后我想到了[chaos-mesh](https://github.com/chaos-mesh)项目中用到的nsexec，这个我以前也用过，是用Rust写的，应该没什么问题也能满足我的需求，而且我也不需要再造轮子了，那么现在的问题就变成了直接改造Litmus的go-runner。

那么我们再一次回到[这里](ttps://github.com/litmuschaos/litmus-go/blob/2.14.1/chaoslib/litmus/stress-chaos/helper/stress-helper.go#L124)。只是这次我把它更新成了
```go
stressCommand := fmt.Sprintf("pause nsexec -m /proc/%d/ns/mnt -c /proc/%d/ns/cgroup -i /proc/%d/ns/ipc -n /proc/%d/ns/net -p /proc/%d/ns/pid -l", targetPID, targetPID, targetPID, targetPID, targetPID) + " -- " + stressors
```
然后我们将`nsexec`和`libnsenter.so`与go-runner代码一起编译打包成docker image。你可能需要从[ns-exec](https://github.com/chaos-mesh/nsexec)的release中找到已经发布的版本。

## 测试
再次启动那个helper pod，只是这次我将`LIB_IMAGE`这个环境变量改成了我已经打包好的image。然后再次运行pod-io-stress，这次就没什么问题了。能从监控指标中看到IO Throughput在不停的上升。而且目标的pod中的容器并没有stress-ng命令行工具，也可以正常的使用。

## 总结
源代码依然是最可靠的东西。Litmus的pod-stress-io看来一直就没可用过，就因为这个问题。但是在知道了原因后解决起来也不是什么很困难的事。了解操作系统的一些知识目前来看，对于测试开发人员来说还是必不可少的。

相关的代码都可以在我的repo中找到，欢迎直接fork。

## 参考资料
[litmuschaos/litmus-go](https://github.com/litmuschaos/litmus-go)

[litmuschaos/test-tools/](https://github.com/litmuschaos/test-tools/)

[runc/nsenter](https://github.com/opencontainers/runc/blob/main/libcontainer/nsenter)

[chaos-mesh/nsexec](https://github.com/chaos-mesh/nsexec)

[一篇关于这个的知乎文章](https://zhuanlan.zhihu.com/p/387830848)

[stackoverflow相关问题](https://stackoverflow.com/questions/31757509/linux-different-threads-of-a-process-in-different-namespaces)

[相关go issue，2014年就存在的老问题了，估计永远不会修](https://github.com/golang/go/issues/8676)