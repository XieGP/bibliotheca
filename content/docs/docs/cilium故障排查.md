---
title: "记一次cilium网络故障排查"
date: 2022-02-14T08:47:11+08:00
draft: false
---

## 背景

客户业务使用RDS, 即跑在k8s上的数据库. 某日早上9:50分, 客户电联表示数据库无法访问. 

## 现象

在接到用户电话后, 立马对当时的pod状态进行排查, 发现数据库POD均为`2/3` , 说明有1个容器不可用. 通过`kubectl describe`发现有容器的readiness probe失败了有如下事件: 

```go
Events:
  Type     Reason     Age                 From     Message
  ----     ------     ----                ----     -------
  Warning  Unhealthy  <invalid> (x11 over <invalid>)  kubelet  Readiness probe failed: Get http://10.244.3.136:9001/health: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

从上面事件可以看出, kubelet尝试使用http形式的readiness probe, 但是超时失败了. 我们模拟kubelet, 从**pod所在机器**上调用 ****`curl http://10.244.3.136:9001/health`命令, 发现http请求超时:

```go
[root@10-10-40-120 ~]# curl http://10.244.3.136:9001/health
curl: (7) Failed connect to 10.244.3.136:9001; 连接超时
```

另外我们查看失败容器的日志, 发现如下日志, 这说明容器内部访问apiserver超时, 我们同样exec进入容器尝试访问apiserver, 确实无法访问:

```go
W0211 01:12:52.815335       1 mssqlleader.go:286] get leader info failed Get https://10.96.0.1:443/api/v1/namespaces/qfusion-admin/endpoints/pathfinder-29194?timeout=10s: context deadline exceeded.
```

与此同时我们还尝试了几个其他的访问方式, 总结如下:

1. 容器内访问宿主机ip/集群其他节点ip/外部ip均失败. 容器网络和外部网络无法通信
2. 容器内访问本节点容器/集群其他节点容器均成功. 说明容器网络本身正常

合理推测, 该问题产生的原因应该和CNI插件cilium有关. 但我们发现cilium-agent和cilium-operator pod状态均正常.

由于用户是生产环境, 所以优先恢复生产可用, 重启了故障节点cilium-agent. 重启完后所有数据库恢复正常.

## 思路整理

由于物理链路都是通的, 访问不通无非两种情况:

1. cilium依据网络策略, 主动drop
2. 操作系统层面drop

由于**重启cilium-agent可以恢复网络**, 所以在**相信cilium代码没bug的前提下**, 应当排除cilium主动丢包. 那问题聚焦到了操作系统丢包, 因为生产环境已经恢复, 我们没法再去验证故障场景, 只能通过现有的日志来分析当时(9:50)分操作系统发生了哪些事情, 是否有人做了操作.

## 日志排查

我们从三个日志维度入手:

1. 操作系统日志`/var/log/messages`
2. 内核日志`dmesg`
3. 故障组件日志, 通过kibana查看

用户告知我们故障的准确时间是9:50左右, 我们着重查找这个区间附近的日志, 结果发现了一些异常日志:

从操作系统日志发现, 在9:49:03有人操作了systemd restart了某个service. 

从组件日志发现9:49:15秒访问外部服务超时, 由于我们的timeout是10秒, 所以该访问请求是9:49:05秒发起的, 而在9秒前的9:48:54秒还有访问成功的日志.

两个时间点仅相差2秒, 合理推测, 9:49:03的这个systemd reload操作导致了本次事故. 

我们拿着操作系统日志询问客户, 客户表示确实在故障时间点执行过`rpm包升级操作`.

## 尝试复现

引起故障的操作找到了, 接着就是在家里尝试复现这个问题. 通过几次尝试, 发现polkit升级过程中会restart polkit.service, 而polkit关联了tuned, tuned的重启会导致操作系统刷新内核参数. 

## 总结

整个故障的链路可以整理为:

rpm update polkit → systemctl restart polkit → systemctl restart tuned → 重置内核参数文件

通过cilium源码可以发现cilium-agent在调用`loader.Loader.Reinitialize` 时会设置如下内核参数, 其中包括了导致本次问题的`net.ipv4.conf.all.rp_filter`参数. 

```go
// package loader base.go
func (l *Loader) Reinitialize(ctx context.Context, o datapath.BaseProgramOwner, deviceMTU int, iptMgr datapath.IptablesManager, p datapath.Proxy) error {
	...
	sysSettings := []sysctl.Setting{
		{Name: "net.core.bpf_jit_enable", Val: "1", IgnoreErr: true},
		{Name: "net.ipv4.conf.all.rp_filter", Val: "0", IgnoreErr: false},
		{Name: "kernel.unprivileged_bpf_disabled", Val: "1", IgnoreErr: true},
		{Name: "kernel.timer_migration", Val: "0", IgnoreErr: true},
	}
  ...
}
```

这个函数在整个生命周期中有五处调用:

1. policy增加/删除
2. agent初始化
3. kvstore GC
4. config modify event

因为生产上的cilium没有开启kvstore, 所以在整套系统平稳运行即无policy增删, 无cilium-agent重启, 无agent配置修改时, 外部触发的内核参数修改行为cilium既无法感知也不会去重置. 这也解释了为什么重启cilium-agent后系统会恢复访问.

## 原理分析

cilium tunnel模式下, 网络包从容器内出来, 先走到veth pair在宿主机上的这端即`lxcxxxxxx` ,根据目标地址分为两种情况:

1. 访问容器网络的, 即dst⊆POD CIDR, 此时根据内核路由, 流量将送往cilium_host接口
2. 访问容器外部网络的, 即dst∉POD CIDR, 此时根据内核路由, 流量将送往对应的接口

rp_filter参数用于开启反向路由检查, 有三个级别, 而操作系统默认参数值为1:

0：不开启源地址校验。
1：开启严格的反向路径校验。对每个进来的数据包，通过路由表校验其反向路径是否是最佳路径。如果反向路径不是最佳路径，则直接丢弃该数据包。
2：开启松散的反向路径校验。对每个进来的数据包，校验其源地址是否可达，即反向路径是否能通（通过任意网口），如果反向路径不同，则直接丢弃该数据包。

**容器访问容器网络(通)**

该访问场景下, src和dst都属于POD CIDR, 即反向路径为最佳路径, 通过参数校验.

数据链路为: 

**container eth0→宿主机 lxcabcd→宿主机 cilium_host→宿主机任意出口流量/pod在本地直接路由** 

**容器访问外部网络(不通)**

该访问场景下, src和dst分属于不同网段, 即反向路径并非最佳路径, 当rp_filter为1时不通过参数校验, 内核将直接丢包. 所以整个数据链路在veth这边就断掉了.

数据链路为:

**container eth0→宿主机 lxcabcd(丢包)**

**外部访问容器网络**

该场景需要分开讨论, 一个是访问当前节点上的POD(不通), 一个是访问其他节点的POD(通). 根据cilium的tunnel模式的数据链路, 分析如下: 

1. 宿主机访问: src是宿主机ip, 例如kubelet的nodeip, 该ip的接口并非cilium_host, **流量可以到达容器内, 但回包会被veth lxc丢弃**, 原理同**容器访问外部网络**的场景一样, 具体就不展开分析了. 
2. 其他节点访问: 对于容器而言, 该场景下src是cilium_host的ip, 因为流量从外部节点过来时需要经过tunnel的封包解包, 该链路是可以访问的.
  
    数据链路为:
    
    **node A cilium_host(封包UDP)→nodeA default interface→nodeB interface→nodeB cilium_host(解包UDP)→nodeB lxc**
    
    很明显可以看到, 在这个场景下对于lxc而言最佳路径即cilium_host, 所以反向路由检查通过, 链路畅通.