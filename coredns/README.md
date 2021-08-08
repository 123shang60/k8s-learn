# CoreDNS

## 进入 bash

`CoreDNS` 默认情况下是没有 `bash` 或者 `sh` 环境的，不能简单的通过 `kubectl exec` 命令进入 `container` 内部。 参考 [How to get into CoreDNS pod kuberrnetes?](https://stackoverflow.com/questions/60666170/how-to-get-into-coredns-pod-kuberrnetes) 方案，可以通过以下步骤进入：

1. 执行 `kubectl get pods -n kube-system` 命令，找到 `coredns` 的 `pod` ，并通过 `kubectl describe po` 命令找到 `pod` 所在的物理机器以及 `docker container id`。
2. ssh 登陆到对应的机器上，执行如下命令：
    ```shell
    ID=<paste ID here>
    docker run -it --net=container:$ID --pid=container:$ID --volumes-from=$ID alpine sh
    ```

## 生效原理

在 `kube-system` 命名空间下，有一个名为 `coredns` 的 `ConfigMap`。在 `CoreDNS` 运行期间，会定时检测此 cm 的变化情况，一旦配置发生变化就会重新加载相关配置。

使用 [进入bash](#进入-bash) 的方法进入 `CoreDNS` 内部，可以发现，`CoreDNS` 本质上是通过拉取并重建软连接加载的方式进行文件替换，从而实现配置文件的变更的。

```bash
/etc/coredns # ll
total 12
drwxrwxrwx    3 root     root          4096 Aug  6 11:57 .
drwxr-xr-x    1 root     root          4096 Aug  8 06:35 ..
drwxr-xr-x    2 root     root          4096 Aug  6 11:57 ..2021_08_06_11_57_41.752147119
lrwxrwxrwx    1 root     root            31 Aug  6 11:57 ..data -> ..2021_08_06_11_57_41.752147119
lrwxrwxrwx    1 root     root            15 Aug  6 11:57 Corefile -> ..data/Corefile
/etc/coredns # 
```

可以明显看到，`CoreDNS` 会将配置按照日期放到 `/etc/coredns` 目录下，并通过软连接挂载到 `/etc/coredns/..data` 目录上。 `Corefile` 本质上则是指向 `/etc/coredns/..data/Corefile` 的软连接，这样就可以做到配置文件的动态替换。

## 自定义解析

参考 [CoreDNS Hosts](https://coredns.io/plugins/hosts/) 配置说明，我们可以通过在 coredns 中增加 hosts 配置的方式，为整个集群增加公共域名解析。

example:
```config
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    hosts {
        1.1.1.1 aaa.com

        fallthrough
    }
    prometheus :9153
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

## 上游 dns 服务器

参考 [CoreDNS Forward](https://coredns.io/plugins/forward/) 配置说明，我们可以为集群配置上游 dns 服务器。

```config
forward . 192.168.90.1 /etc/resolv.conf {
    max_concurrent 1000
}
```

> 注意: `CoreDNS` 的默认解析策略为在多个 dns 中进行轮询。因此，如果某个域名仅在某一个上游 dns 中存在，而不在其他 dns 中存在时，会导致解析异常。

## K8S DNS 策略

在 k8s 中，存在以下几种 DNS 策略。

- ClusterFirstWithHostNet  
  当 `POD` 以 `HOST` 模式进行启动时，会与主机共享网络配置，此时将不会通过 k8s 内部的 `coredns` 进行解析。如果在这种情况下仍想让对应的 `PDO` 解析 K8S 内部的域名，就需要设置为此种 DNS 策略
- ClusterFirst  
  此设置为 `POD` 的默认设置，会要求 `POD` 优先使用 k8s 内部的域名解析，在解析失败的情况下才使用宿主机的网络配置进行解析
- Default  
  这种情况下，`POD` 会根据 `kubelet` 的要求进行域名解析。默认情况下，`kubelet` 使用本机的 dns 配置文件进行解析。可以通过调整 `kubelet` 的启动参数 `--resolv-conf=/etc/resolv.conf` 来控制 `kubelet` 默认使用的 DNS 配置文件
- None  
  这种配置一般是在 `dnsConfig` 自定义 `POD` 域名解析策略时使用。否则，此配置不可用

## 参考文献

- [coredns](https://coredns.io)
- [kubernetes-dns](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)