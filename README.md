---
title: Kubernetes Pod
tags: kubernetes, k8s, Pod, Static Pod
---

# Kubernetes Pod

## Pod創建過程及種類
本篇介紹一些Pod的進階概念，首先要了解Pod的創建過程為何。

:::info
當一個Pod被創立時，是由Master 先建立 pod-def.yaml file(或透過command)，透過kube-apiserver命令Slave node中的kubelet，kubelet透過啟動一個名為```k8s.gcr.io/pause:3.1``` 的image來創建pod。
![](https://i.imgur.com/1aWtZUt.png)


**P.S. pod及container名稱不能用大寫**
:::

但是Pod創建方式其實不只上述一種，舉例來說，假設這個cluster只有一個Slave node，既沒有Master，也沒有第二個Slave node，是個單一節點的叢集。那麼，這個叢集即無法透過kube-apiserver通知kubelet創建Pod(因為沒有Master)。所以有另一種創建Pod的方式

:::info
將pod-def.yaml file放在特定位置下，由kubelet直接根據這個位置的yaml file來創建Pod，用這種方式創建的Pod，稱為static pod
:::

kubelet會有兩種input
*    透過特定路徑下的 pod-def.yaml file，創建static pods
*    Master利用kube-apiserver並通過HTTP API ebdpoint的方式

所以，Pod依創建方式可分為兩種
*    static pods
*    kube-apiserver

當kubelet創建static pod時，會在同叢集的kube-apiserver中創建一個新的mirror object。可透過kubectl 觀察這個Pod，但是無法操作(修改、刪除)，這是因為kubectl指令必須搭配kube-apiserver使用，而這時看到的Pod只是static pod的mirror object

## 為何需要static pod ? 
因為static pod不需依靠controller plane的物件，所以可以透過static pod創建屬於自己node中的controller plane物件，例如，Master中的controller.yaml, etcd.yaml等等，其實都算是一種static pod。還有另一好處，就是不用擔心controller plane server cashing，而 static pod 若發生crash，kubelet會自動restart一個新的

**這就是為甚麼在Master上使用kubectl get po --namespace=kube-system時，看到的所有物件都是Pod的形式，這些就是static pod。Master節點上的物件基本上都是以static pod的方式創建，畢竟Master若crash掉整個cluster基本上就掛了**

當使用 **kubectl get po --all-namespace** 時，要如何得知那些Pod是static pod，那些不是呢? 其實很簡單，只要Pod name後面跟著 **-master** 的，就都是static pod

---

## 如何知道哪個路徑用來創建static pod

配置文件就是放在特定目錄下的標準的 yaml 格式的 Pod 定義文件。用 **kubelet --pod-manifest-path =** 來啟動kubelet，kubelet 定期的去掃描這個目錄，根據這個目錄下出現或消失的 yaml 文件來創建或刪除static pod，那麼，要如何知道這個目錄在哪裡呢，可以通過以下指令 **ps -aux | grep kubelet** 

(也可先進入/etc/systemd/system/kubelet.service.d目錄下查看10-kubeadm.conf這個檔案)
```bash=
$ ps -aux | grep kubelet
root        665  3.9  1.5 1705920 64016 ?       Ssl  00:50  50:30 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --pod-infra-container-image=k8s.gcr.io/pause:3.1 --resolv-conf=/run/systemd/resolve/resolv.conf
root       3100  6.3  6.2 496088 248664 ?       Ssl  00:50  81:37 kube-apiserver --advertise-address=192.168.17.131 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
george    15571  0.0  0.0  21532  1060 pts/0    S+   22:15   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn kubelet

```
然後，找到 **--config=/var/lib/kubelet/config.yaml** 這個參數，並且檢查 config.yaml 這個檔案

```bash=
$ cat /var/lib/kubelet/config.yaml 
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s

```

找到這一行
```=
staticPodPath: /etc/kubernetes/manifests
```
這個路徑就是創建static pod的特定目錄啦~

---

這邊有個觀念很重要
:::warning
直接用**kubectl delete po \<static-pod-name\>** 指令刪除static pod 是無法成功的，因為 kubelet 會定期檢查特定目錄下的檔案，即使static pod被刪除，但只要其 yaml file 還存在，kubelet就會重新創建一個新的static pod。若要刪除static pod，需刪除其目錄下的 yaml file
:::

## 範例

舉個例子，現在我們要刪除static-greenbox-node01這個pod

```bash=
master $ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
static-greenbox-node01   1/1     Running   0          104s
```
我們到 **/etc/kubernetes/manifests** 目錄下找這個file
```bash=
master $ pwd
/etc/kubernetes/manifests
master $ ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```
發現找不到這個file，那這個static pod是如何創建的呢? 我們 **get pod -o wide** 看看
```bash=
master $ kubectl get po -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE     NOMINATED NODE   READINESS GATES
static-greenbox-node01   1/1     Running   0          6m50s   10.44.0.1   node01   <none>           <none>
```
發現他在node01這個節點上，這時我們ssh進入node01 ( **ssh \<node-ip-address\>** )
進來後一樣使用 **ps -aux | grep kubelet** 查找，然後查看 **/var/lib/kubelet/config.yaml** 檔案


```bash=
node01 $ cat /var/lib/kubelet/config.yaml
address: 0.0.0.0
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: cgroupfs
cgroupsPerQOS: true
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuCFSQuotaPeriod: 100ms
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kind: KubeletConfiguration
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusReportFrequency: 1m0s
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/just-to-mess-with-you
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
topologyManagerPolicy: none
volumeStatsAggPeriod: 1m0s
```

赫然發現 static pod的創見目錄變更了 位於 **staticPodPath: /etc/just-to-mess-with-you** ，索性進入該目錄
```bash=
node01 $ pwd
/etc/just-to-mess-with-you
node01 $ ls
greenbox.yaml
```
真的發現這個 static pod file 了，將其刪除就大功告成啦~

---

## 創建 static pod

在knode上創建static pod，先找到創建的特定目錄，在該目錄下創建static-pod.yaml，使用最簡單的nginx image做範例，內容如下：
```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
```
但是，若直接將此 static pod 創建起來，會一直進入CrashBackOff狀態，需要重新config你的kubelet才可以。
輸入 **KUBELET_ARGS="--cluster-dns=10.254.0.10 --cluster-domain=kube.local --pod-manifest-path=\<static-pod-dic\>"**

```bash=
KUBELET_ARGS="--cluster-dns=10.254.0.10 --cluster-domain=kube.local --pod-manifest-path=/etc/kubernetes/manifest"
```
接著重啟kubelet
```bash=
systemctl restart kubelet
```
到Master查看，就發現 Pod 處於 Running status囉～
```bash=
$ kubectl get po
NAME               READY   STATUS    RESTARTS   AGE
static-web-knode   1/1     Running   0          3h31m
```
---
## 如何看Pod網域
當查看Pod時，可以看到它們的IP addr
```bash=
root@gmaster:/home/gmaster/k8s_demo/1-basicObject# kubectl get no -o wide
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
gmaster   Ready    master   9d    v1.17.4   10.10.14.124   <none>        Ubuntu 18.04 LTS     4.15.0-20-generic   docker://19.3.6
gnode01   Ready    <none>   9d    v1.17.4   10.10.11.10    <none>        Ubuntu 18.04 LTS     4.15.0-20-generic   docker://19.3.6
gnode02   Ready    <none>   9d    v1.17.4   10.10.19.6     <none>        Ubuntu 18.04.4 LTS   4.15.0-20-generic   docker://19.3.6
```
可以發現它們的IP都是10.10.X.X，為甚麼他們的IP都有相同的的mask呢?又要如何確定它們的mask是多少?
首先，先觀察它們是用哪張interface。可以用```arp```來查看
```bash=
oot@gmaster:/home/gmaster/k8s_demo/1-basicObject# arp
Address                  HWtype  HWaddress           Flags Mask            Iface
gnode01                  ether   00:50:56:b5:58:1d   C                     ens160
172.17.0.4               ether   02:42:ac:11:00:04   C                     docker0
172.17.0.2               ether   02:42:ac:11:00:02   C                     docker0
169.254.169.254                  (incomplete)                              ens160
172.17.0.3               ether   02:42:ac:11:00:03   C                     docker0
10.10.18.186             ether   ac:f1:df:79:28:1c   C                     ens160
gnode02                  ether   00:50:56:b5:d6:93   C                     ens160
dlink.svc                ether   00:18:e7:da:f7:0b   C                     ens160
dns.h                    ether   00:11:32:2c:a6:03   C                     ens160
10.10.10.8               ether   d0:bf:9c:bd:42:f4   C                     ens160
10.10.10.9               ether   c8:d9:d2:cb:f2:3a   C                     ens160
```
可以發現gmaster是使用**ens160**這張interface，再用```ip addr```指令，並觀察**ens160**這張interface的資訊
```bash=
root@gmaster:/home/gmaster/k8s_demo/1-basicObject# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b5:66:81 brd ff:ff:ff:ff:ff:ff
    inet 10.10.14.124/16 brd 10.10.255.255 scope global dynamic noprefixroute ens160
       valid_lft 567sec preferred_lft 567sec
    inet6 2002:8c73:3b40:100:1fa:4813:8b09:4316/64 scope global temporary dynamic 
       valid_lft 65533sec preferred_lft 20150sec
    inet6 2002:8c73:3b40:100:c9e1:229f:d24b:7312/64 scope global temporary deprecated dynamic 
       valid_lft 65533sec preferred_lft 0sec
    inet6 2002:8c73:3b40:100:514d:7e04:c3ff:3602/64 scope global temporary deprecated dynamic 
       valid_lft 65533sec preferred_lft 0sec
    inet6 2002:8c73:3b40:100:5ca8:c52c:4145:664f/64 scope global temporary deprecated dynamic 
       valid_lft 65533sec preferred_lft 0sec
    inet6 2002:8c73:3b40:100:28cb:33e9:d27:a1f0/64 scope global temporary deprecated dynamic 
       valid_lft 65533sec preferred_lft 0sec
    inet6 2002:8c73:3b40:100:1468:1c8d:c76f:c936/64 scope global temporary deprecated dynamic 
       valid_lft 65533sec preferred_lft 0sec
    inet6 2002:8c73:3b40:100:2865:651d:5726:f3ad/64 scope global temporary deprecated dynamic 
       valid_lft 21467sec preferred_lft 0sec
    inet6 2002:8c73:3b40:100:a06a:9f79:4918:92ab/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 65533sec preferred_lft 65533sec
    inet6 fe80::739b:d560:7283:502e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:31:3b:d7:f7 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:31ff:fe3b:d7f7/64 scope link 
       valid_lft forever preferred_lft forever
5: vethd3da50e@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 9e:93:c6:8d:47:1c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::9c93:c6ff:fe8d:471c/64 scope link 
       valid_lft forever preferred_lft forever
7: veth8440c63@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether ea:5a:ce:43:bf:d0 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::e85a:ceff:fe43:bfd0/64 scope link 
       valid_lft forever preferred_lft forever
38: datapath: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 82:e7:d4:de:dc:4c brd ff:ff:ff:ff:ff:ff
    inet6 fe80::80e7:d4ff:fede:dc4c/64 scope link 
       valid_lft forever preferred_lft forever
40: weave: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default qlen 1000
    link/ether 2e:66:9c:b0:1c:ca brd ff:ff:ff:ff:ff:ff
    inet 10.44.0.0/12 brd 10.47.255.255 scope global weave
       valid_lft forever preferred_lft forever
    inet6 fe80::2c66:9cff:feb0:1cca/64 scope link 
       valid_lft forever preferred_lft forever
42: vethwe-datapath@vethwe-bridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue master datapath state UP group default 
    link/ether 02:6a:73:be:b5:0a brd ff:ff:ff:ff:ff:ff
    inet6 fe80::6a:73ff:febe:b50a/64 scope link 
       valid_lft forever preferred_lft forever
43: vethwe-bridge@vethwe-datapath: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue master weave state UP group default 
    link/ether 22:06:2f:77:5b:cf brd ff:ff:ff:ff:ff:ff
    inet6 fe80::2006:2fff:fe77:5bcf/64 scope link 
       valid_lft forever preferred_lft forever
44: vxlan-6784: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65535 qdisc noqueue master datapath state UNKNOWN group default qlen 1000
    link/ether ea:25:df:20:5c:f6 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::e825:dfff:fe20:5cf6/64 scope link 
       valid_lft forever preferred_lft forever
```
**ens160**下顯示```inet 10.10.14.124/16```，就可以看出所有使用ens160 interface的Pod IP addr 都會是```10.10.x.x/16```這個prefix。



---


## Pod 操作
創建Pod
```bash=
kubectl run <pod-name> --image=<image> --generator=run-pod/v1 --restart=Never --dry-run=client -o yaml > pod-def.yaml
```
:::success
```--generator=```其實也可以用來創建其他object，但是[官方文獻](https://kubernetes.io/docs/reference/kubectl/conventions/)不推薦用來創建Pod以外的物件；

```restart=Never```用來識別Pod，若```restart=Always```則是deployment；

```--dry-run```表示一些default的參數先拿掉，暫時不需要submit出去 (通常在輸出yaml時使用)
:::

查看Pod label
```=
kubectl get pod --show-labels
```
新增Pod label
```=
kubectl label po <pod-name> <key>=<value>
```

刪除Pod label
```=
kubectl label po <pod-name> <key>-
```
對Pod下內部指令
```=
kubectl exec <pod-name> -- <command>
```
將Pod expose出去 **(創建一個service)**

```=
kubectl expose po <pod-name> --type=NodePort --name=<svc-name> --port=80
```




---

### Thank you! :sheep: 

You can find me on

- george4908090@gmail.com