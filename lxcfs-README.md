# Q&A

1. **怎么保障lxcfs daemonset稳定？部署的资源共享kube-system？**

lxcfs daemonset设置了`resource limit`（guaranteed qos），以及设置`priorityClassName`为最高的`system-node-critical`

2. **lxcfs daemonset 重启期间，对业务监控的影响范围有多大？此时业务ps/top/free失败，强依赖监控进行调度的业务，是否有容错能力？**

lxcfs重启期间，访问/proc下的系统接口会失败，对普通业务无影响；需要访问这些接口的监控会立刻失败，需要监控有重试和容错的处理逻辑

3. **node节点重启，如果node上有大量pod挂载lxcfs，会导致很多pod失败？**

如果lxcfs启动前，有使用lxcfs了pod先启动，会导致pod一直处于`ContainerCreating`状态。待lxcfs启动后，会启动成功。
另外需要修改pod中volumes的定义，严格按照这里配置：https://git.code.oa.com/borgerli/lxcfs-image/tree/master#volume

4. **怎么监控lxcfs运行是否异常，日志是否记录？**

lxcfs有日志，可以查看。如果启动有错误，等待一定时间后后（默认2s，lxcfs-ds.yaml中args的第一个参数可以指定等待0.5s的次数）就会退出。如果中途lxcfs因其他原因意外退出，lxcfs pod也会退出。可以从lxcfs pod的重启来判断。
另外可以在lxcfs-ds.yaml中args增加第三个参数"-d"，开启debug模式

5. **lxcfs root路径变化了，存量使用lxc的pod，怎么平滑切换？**

lxcfs-ds.yaml中args的第二个参数可以指定lxcfs root。但是这个没有修改的必要，另外修改了pod肯定也要修改volume定义，没法平滑。

# 部署lxcfs

## Docker

```
docker run -it --privileged -v /var/lib/lxc/lxcfs:/var/lib/lxc/lxcfs:rshared --pid host ccr.ccs.tencentyun.com/tkeimages/lxcfs:4.0.8
```

## K8s

```
kubectl apply -f lxcfs-ds.yaml
```
# 测试

## Docker
```
docker run -it -m 128m --cpu-period 100000 --cpu-quota 100000 --rm \
  -v /var/lib/lxc/lxcfs/proc/cpuinfo:/proc/cpuinfo:rw \
  -v /var/lib/lxc/lxcfs/proc/diskstats:/proc/diskstats:rw \
  -v /var/lib/lxc/lxcfs/proc/meminfo:/proc/meminfo:rw \
  -v /var/lib/lxc/lxcfs/proc/stat:/proc/stat:rw \
  -v /var/lib/lxc/lxcfs/proc/swaps:/proc/swaps:rw \
  -v /var/lib/lxc/lxcfs/proc/loadavg:/proc/loadavg:rw \
  -v /var/lib/lxc/lxcfs/proc/uptime:/proc/uptime:rw \
  -v /var/lib/lxc/lxcfs/sys/devices/system/cpu/online:/sys/devices/system/cpu/online:rw \
  -v /var/lib/lxc:/var/lib/lxc:rshared \
  centos:7 /bin/bash
```

## K8s

给业务负载的Pod Spec增加如下volume信息以及给Pod Spec的container增加如下volumeMount信息
### volume
```
      volumes:
      - name: cpuinfo
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/cpuinfo
          type: File
      - name: diskstats
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/diskstats
          type: File
      - name: loadavg
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/loadavg
          type: File
      - name: meminfo
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/meminfo
          type: File
      - name: stat
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/stat
          type: File
      - name: swaps
        hostPath:
          path:  /var/lib/lxc/lxcfs/proc/swaps
          type: File
      - name: uptime
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/uptime
          type: File
      - name: cpu-online
        hostPath:
          path: /var/lib/lxc/lxcfs/sys/devices/system/cpu/online
          type: File
      - name: lxcfs
        hostPath:
          path: /var/lib/lxc
          type: Directory
```
### volumeMounts
```
        volumeMounts:
        - name: cpuinfo
          mountPath: /proc/cpuinfo
        - name: diskstats
          mountPath: /proc/diskstats
        - name: loadavg
          mountPath: /proc/loadavg
        - name: meminfo
          mountPath: /proc/meminfo
        - name: stat
          mountPath: /proc/stat
        - name: swaps
          mountPath: /proc/swaps
          readOnly: true
        - name: uptime
          mountPath: /proc/uptime
        - name: cpu-online
          mountPath: /sys/devices/system/cpu/online
        - name: lxcfs
          mountPath: /var/lib/lxc
          mountPropagation: HostToContainer
```

### 完整示例
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lxcfs-test
  labels:
    app: lxcfs-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lxcfs-test
  template:
    metadata:
      labels:
        app: lxcfs-test
    spec:
      containers:
      - name: nginx
        image: ubuntu:20.04
        command: ["/bin/bash"]
        args: ["-c","while true; do date && sleep 60;done"]
        resources:
          limits:
            cpu: 1
            memory: 64Mi
        volumeMounts:
        - name: cpuinfo
          mountPath: /proc/cpuinfo
        - name: diskstats
          mountPath: /proc/diskstats
        - name: loadavg
          mountPath: /proc/loadavg
        - name: meminfo
          mountPath: /proc/meminfo
        - name: stat
          mountPath: /proc/stat
        - name: swaps
          mountPath: /proc/swaps
          readOnly: true
        - name: uptime
          mountPath: /proc/uptime
        - name: cpu-online
          mountPath: /sys/devices/system/cpu/online
        - name: lxcfs
          mountPath: /var/lib/lxc
          mountPropagation: HostToContainer
      volumes:
      - name: cpuinfo
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/cpuinfo
          type: File
      - name: diskstats
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/diskstats
          type: File
      - name: loadavg
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/loadavg
          type: File
      - name: meminfo
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/meminfo
          type: File
      - name: stat
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/stat
          type: File
      - name: swaps
        hostPath:
          path:  /var/lib/lxc/lxcfs/proc/swaps
          type: File
      - name: uptime
        hostPath:
          path: /var/lib/lxc/lxcfs/proc/uptime
          type: File
      - name: cpu-online
        hostPath:
          path: /var/lib/lxc/lxcfs/sys/devices/system/cpu/online
          type: File
      - name: lxcfs
        hostPath:
          path: /var/lib/lxc
          type: Directory
```

## 验证

业务容器启动后，进入容器执行`free -m`，`grep processor /proc/cpuinfo | wc -l`验证内存及cpu信息。

