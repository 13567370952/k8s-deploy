# 离线安装 kubernetes 1.5 高可用集群

经常遇到全新初始安装k8s集群的问题，所以想着搞成离线模式，本着最小依赖原则，采用纯shell脚本编写

基于Centos7-1503-minimal运行脚本测试OK， 默认安装docker1.12.3 etcd-v3.0.15 k8s-v1.5.1

本离线安装所有的依赖都打包放到了[百度网盘](https://pan.baidu.com/s/1i5jusip)

简要说明

* 基于kubeadm搭建的kubernetes1.5 HA高可用集群
* HA环境，需要先存在etcd集群，可使用etcd目录下的一键部署etcd集群脚本
* 默认master和etcd共用一台设备，共三台相互冗余，自行确保所有设备开启NTP同步
* master间通过keepalived做主-从-从冗余， controller和scheduler通过自带的--leader-elect选项
* 如果只想部署单master的话， 可以修改脚本里KUBE_HA=false
* 如果想部署kubeadm的默认模式，即全面容器化单实例启动，可以参考[这里](https://github.com/xiaoping378/blog/issues/5)
* 现在的keepalived和etcd集群没用容器运行，后面有时间会尝试做到全面容器化

## 第一步
离线安装的基本思路是，在k8s-deploy目录下，临时启个http server， 节点上会从此拉取所依赖镜像和rpms

```
# python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
```

windows上可以用hfs临时启个http server， 自行百度如何使用

## master侧

运行以下命令，初始化master， master侧如果是单核的话，会因资源不足， dns安装失败。

```
curl -L http://192.168.56.1:8000/k8s-deploy.sh | bash -s master \
    --api-advertise-addresses=192.168.56.103 \
    --external-etcd-endpoints=http://192.168.56.100:2379,http://192.168.56.101:2379,http://192.168.56.102:2379
```

* **192.168.56.1:8000** 是我的http-server, 注意要将k8s-deploy.sh 里的HTTP-SERVER变量也改下

* **--api-advertise-addresses** 是VIP地址

* **--external-etcd-endpoints** 是你的etcd集群地址，这样kubeadm将不再生成etcd.yaml manifest文件

* 记录下你的token输出， minion侧需要用到

* 安装docker时，如果之前装过“不干净的”东西，可能会遇到依赖问题，我这里会遇到systemd-python依赖问题，
卸载之，即可
```yum remove -y systemd-python```

## replica master侧

在replica master侧运行下面的命令，会自动和第一个master组成冗余

最好和第一个master建立免秘钥认证，此过程需要从master那里拷贝配置
```
curl -L http://192.168.56.1:8000/k8s-deploy.sh | bash -s replica \
    --api-advertise-addresses=192.168.56.103 \
    --external-etcd-endpoints=http://192.168.56.100:2379,http://192.168.56.101:2379,http://192.168.56.102:2379
```

重复上面的步骤之后，会有一个3实例的HA集群，执行下面命令的时候可关闭第一个master，以验证高可用

* 验证vip漂移的网络影响

    ping 192.168.56.103

* 验证kube-apiserver故障影响

  1.5.1，默认关闭了匿名访问，要通过带token的方式访问API
  ```
  while true; do kubectl get po -n kube-system; sleep 1; done
  ```

## minion侧

视自己的情况而定， 使用第一个master侧生成的token

```
curl -L http://192.168.56.1:8000/k8s-deploy.sh |  bash -s join --token=6c96b6.ca1d13745c237904 192.168.56.103
```

## 总结

整个脚本实现比较简单， 坑都在脚本里解决了。
就一个master-up和node-up， 基本一个函数只做一件事，很清晰，可以自己查看具体过程。

1.5 与 1.3给我感觉最大的变化是网络部分， 1.5启用了cni网络插件
不需要像以前一样非要把flannel和docker绑在一起了（先启flannel才能启docker）。

具体可以看这里
https://github.com/containernetworking/cni/blob/master/Documentation/flannel.md
