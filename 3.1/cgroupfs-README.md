# cgroupfs挂载方式

## 隔离视图
当前实现的典型资源视图隔离支持如下：

- /proc/cpuinfo
- /proc/stat
- /proc/meminfo
- /proc/loadavg
- /proc/diskstats
- /proc/uptime


## 节点挂载cgroupfs
```
$ mkdir /cgroupfs

$ mount -t cgroupfs cgroupfs /cgroupfs/
```

TKE 新加节点可以走自定义数据方式实现，参考：https://cloud.tencent.com/document/product/457/32206

# Bind mount实现资源隔离

以 meminfo 为例，完整示例如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: nginx
    qcloud-app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: nginx
      qcloud-app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: nginx
        qcloud-app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: IfNotPresent
        name: nginx
        resources:
          limits:
            memory: 1Gi
          requests:
            memory: 1Gi
        securityContext:
          privileged: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /proc/meminfo
          name: meminfo
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /cgroupfs/proc/meminfo
          type: File
        name: meminfo
```
