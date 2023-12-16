# 使用Litmus pod-stress相关的故障发现的奇怪问题
## 背景
失业了，没事只能在家开发故障框架，想直接把一些Litmus的故障加入进来吧，这样好快速达成我的目的。

一开始还好，operator项目也快速的起来了，抛弃了之前一直用来存数据的MySQL，这下全云原生了算是，后边在自己的colima+k3s+docker的环境里一跑，诡异的就来了，根本就起不来了。心想不对吧，之前在公司里还跑过妹问题啊，那么开始分析时间。

## 分析
看到的日志里出现这么一条。

`time="2023-02-17T09:16:37Z" level=fatal msg="helper pod failed, err: fail to get the cgroup manager, err: Error loading cgroup v2 manager, cgroups: invalid group path"`

看样子是cgroup2被启用了？那么到qemu Linux虚拟机中再确认一下，果然诶
```shell
$ grep cgroup /proc/filesystems
nodev   cgroup
nodev   cgroup2
```

那么就没问题了，就是cgroup2相关的代码逻辑里有问题了。然后就开始搜索stress helper里相关的代码，找到这行日志报错的地方。我用的是2.14的代码，所以直接切了v2.14.x的分支了，大概就是在stress-helper.go的468行里有这么一段。
```go
if cgroups.Mode() == cgroups.Unified {
	groupPath, err := cgroupsv2.PidGroupPath(pid)
	if err != nil {
		return nil, errors.Errorf("Error in getting groupPath, %v", err)
	}

	cgroup2, err := cgroupsv2.LoadManager("/sys/fs/cgroup", groupPath)
	if err != nil {
		return nil, errors.Errorf("Error loading cgroup v2 manager, %v", err)
	}
	return cgroup2, nil
}
```
这个`cgroupsv2.LoadManager`函数我检查了一下，最后其实也就是把这两个路径拼起来而已，那我只能猜这个路径有问题了。那么再检查这个`groupPath`是怎么得到的了。

一路检查上边的`cgroupsv2.PidGroupPath`的函数到`containerd/cgroups`包里，最终应该是这样的。
```go
func parseCgroupFromReader(r io.Reader) (string, error) {
	var (
		s = bufio.NewScanner(r)
	)
	for s.Scan() {
		var (
			text  = s.Text()
			parts = strings.SplitN(text, ":", 3)
		)
		if len(parts) < 3 {
			return "", fmt.Errorf("invalid cgroup entry: %q", text)
		}
		// text is like "0::/user.slice/user-1001.slice/session-1.scope"
		if parts[0] == "0" && parts[1] == "" {
			return parts[2], nil
		}
	}
	if err := s.Err(); err != nil {
		return "", err
	}
	return "", fmt.Errorf("cgroup path not found")
}
```
就和注释里写的一样，最终应该是返回了`:`分割后的slice的第2个index的字符串，前提条件是这玩意是我们预期得到的东西。按照这个代码里的逻辑，我又去查看了/proc/\<pid>/cgroup里的内容，在容器下我看到的就不对了，返回的内容就变成了这样的。
```
root@chaos-79bb494f5-82dz5:/litmus# cat /proc/5071/cgroup 
0::/../../../podd4938737-4c89-4c19-9023-bf251a10ee30/5a860022668d90b18979f4ea5da843dee065db24133e5f3a14b4e0dbb03c8835
```
很显然，最终我们得到的这个`groupPath`的结果其实是
```
/../../../podd4938737-4c89-4c19-9023-bf251a10ee30/5a860022668d90b18979f4ea5da843dee065db24133e5f3a14b4e0dbb03c8835
```

但如果我直接在host的shell环境下，看到的却是
```
0::/kubepods/podd4938737-4c89-4c19-9023-bf251a10ee30/5a860022668d90b18979f4ea5da843dee065db24133e5f3a14b4e0dbb03c8835
```

然后我仔细看了一遍Linux Kernel里关于cgroup v2的相关文档，看到一段是这么描述的。
```
From a sibling cgroup namespace (that is, a namespace rooted at a different cgroup), the cgroup path relative to its own cgroup namespace root will be shown. For instance, if PID 7353's cgroup namespace root is at '/batchjobs/container_id2', then it will see:

# cat /proc/7353/cgroup
0::/../container_id2/sub_cgrp_1
```
恍然大悟了，难怪我看不到绝对路径了。那么这问题就简单了，我直接切换到root namespace不就行了。于是在容器中这样跑
```
root@chaos-79bb494f5-82dz5:/litmus# nsenter -t 1 -C -- cat /proc/5071/cgroup 
0::/kubepods/podd4938737-4c89-4c19-9023-bf251a10ee30/5a860022668d90b18979f4ea5da843dee065db24133e5f3a14b4e0dbb03c8835
```
这样我们就得到了一个正确的绝对路径。
修复完后，我又重新跑了一遍，还是有错，但是这次一样了，变成了
```
fail to add the stress process into target container cgroup, err: Permission Denied
```
大致上是这个错误，可能没写全，不过就是这个意思，没权限去写入那个文件。那么到这里，再次进入源代码检查。

## 挂载进目标cgroup的问题
还是一样的stress-helper.go，但是这次去了146行的位置，代码如下
```go
if err = addProcessToCgroup(cmd.Process.Pid, cgroupManager); err != nil {
	if killErr := cmd.Process.Kill(); killErr != nil {
		return errors.Errorf("stressors failed killing %v process, err: %v", cmd.Process.Pid, killErr)
	}
	return errors.Errorf("fail to add the stress process into target container cgroup, err: %v", err)
}
```
点进去后，继续往下深挖，最后到了又回到了`containerd/cgroups`里，其实整个代码不难，就是往刚才找到的cgroup路径下的`cgroup.procs`文件里写入pid，仅此而已。那么好了，我们自己在容器里用shell直接试试看。
```
root@chaos-79bb494f5-82dz5:/litmus# echo 11147 >> /sys/fs/cgroup/kubepods/podd4938737-4c89-4c19-9023-bf251a10ee30/5a860022668d90b18979f4ea5da843dee065db24133e5f3a14b4e0dbb03c8835/cgroup.procs 
bash: echo: write error: No such file or directory
```
按照我之前的经验，盲猜还是因为我并不在host namespace的问题，那么还是有请我们的老朋友`nsenter`
```
root@chaos-79bb494f5-82dz5:/litmus# nsenter -t 1 -C -- sudo sh -c "echo 11147 >> /sys/fs/cgroup%s/cgroup.procs"
```
运行完后，不报错了，那么我们再检查一下是不是真的进去了
```
root@chaos-79bb494f5-82dz5:/litmus# cat /sys/fs/cgroup/kubepods/podd4938737-4c89-4c19-9023-bf251a10ee30/5a860022668d90b18979f4ea5da843dee065db24133e5f3a14b4e0dbb03c8835/cgroup.procs
23890
12483
11147
```
至此，应该问题全部解除。那么将相关的修复代码提交到`stress-helper.go`里，然后重新编译打包image，再跑一次。

最终，我看到了指标数据在目标容器中出现了，确认修复是有效的。

## 参考资料
[cgroup-v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)

[Litmus相关issue](https://github.com/litmuschaos/litmus/issues/3902)

[最终我还是提了个PR修复这个问题](https://github.com/litmuschaos/litmus-go/pull/677)