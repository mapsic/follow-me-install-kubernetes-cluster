
# 10. 部署GlusterFS+Heketi实现Kubernetes共享存储

<!-- TOC -->

- [10. 部署GlusterFS+Heketi实现Kubernetes共享存储](#GlusterFS)
  - [GlusterFS配置](#GlusterFS配置)
    - [安装](#安装)
    - [测试](#测试)
  - [heketi配置](#heketi配置)
    - [简介](#简介)
    - [配置configmap](#配置configmap)
    - [部署到k8s](#部署到k8s)
  - [heketi添加glusterfs](#heketi添加glusterfs)
    - [添加cluster](#添加cluster)
    - [添加device](#添加device)
    - [添加volume](#添加volume)
  - [配置kubernetes使用glusterfs](#配置kubernetes使用glusterfs)
    - [创建storageclass](#创建storageclass)
    - [创建pvc](#创建pvc)
    - [创建pod,并使用pvc](#创建pod,并使用pvc)

<!-- /TOC -->

注意：如下操作，如无特殊说明，均在 **k8s-master01** 节点上操作，
      以下6台机器已经做过SSH免密钥,并配置了hosts解析

## 环境配置如下

``` bash
序号    服务器用途	主机名	        IP地址
1	master节点1	k8s-master01	10.50.41.41
2	master节点2	k8s-master02	10.50.41.42
3	node节点1	k8s-node01	10.50.41.51
4	node节点2	k8s-node02	10.50.41.52
5	共享存储1	k8s-db01	10.50.41.61
6	共存存储2	k8s-db02	10.50.41.62

```

## GlusterFS配置

### yum源配置

``` bash
cd /opt/k8s/work
cat > glusterfs.repo <<'EOF'
[centos-gluster5]
name=CentOS-$releasever - GlusterFS 5
baseurl=https://mirrors.aliyun.com/centos/$releasever/storage/$basearch/gluster-5/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
EOF
```

### 分发yum源配置到node和存储节点

RPM-GPG-KEY-CentOS-SIG-Storage 文件在manifests目录下

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-node01 k8s-node02 k8s-db01 k8s-db02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    scp glusterfs.repo root@${node_name}:/etc/yum.repos.d/glusterfs.repo
    scp RPM-GPG-KEY-CentOS-SIG-Storage root@${node_name}:/etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
  done
```

### 安装server端

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-db01 k8s-db02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    ssh root@${node_name} yum -y install glusterfs-server-5.3-2.el7
  done
```

### 启动服务

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-db01 k8s-db02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    ssh root@${node_name} "systemctl enable glusterd && systemctl restart glusterd"
  done
```

### 在node节点上安装客户端

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-node01 k8s-node02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    ssh root@${node_name} "yum -y install glusterfs-client-xlators-5.3-2.el7 glusterfs-fuse-5.3-2.el7"
  done
```

### 磁盘分区

分别SSH远程到k8s-db01 和k8s-db02 节点上执行下面的分区命令,不用格式化
将/dev/sdb磁盘先分两个1.5T的主分区

``` bash
$ parted /dev/sdb
GNU Parted 3.1
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel msdos
(parted) mkpart primary 0 1500000
Warning: The resulting partition is not properly aligned for best performance.
Ignore/Cancel? ignore
(parted) mkpart primary 1500001  3000001
(parted) p
Model: VMware Virtual disk (scsi)
Disk /dev/sdb: 5498GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      512B    1500GB  1500GB  primary
 2      1500GB  3000GB  1500GB  primary

(parted) quit
Information: You may need to update /etc/fstab.
```

### 配置可信任的存储池

``` bash
cd /opt/k8s/work
ssh root@k8s-db01 "gluster peer probe k8s-db02"  
```

预期输出

``` bash
>>> k8s-db01

peer probe: success.
```

一旦建立了该池，只有受信任的成员才可以将新服务器探测到池中。 新服务器无法探测池，必须从池中探测。

检查对端状态

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-db01 k8s-db02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    ssh root@${node_name} "gluster peer status"
  done
```

预期输出

``` bash
>>> k8s-db01
Number of Peers: 1

Hostname: k8s-db02
Uuid: ea50a5e4-f3fd-47ff-b356-fd4e636aae8d
State: Peer in Cluster (Connected)
>>> k8s-db02
Number of Peers: 1

Hostname: k8s-db01
Uuid: f50a0a61-004e-4a74-bf68-6c928c4400c2
State: Peer in Cluster (Connected)
```

## 测试

### 创建测试卷

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-db01 k8s-db02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    ssh root@${node_name} "mkdir -p /home/glusterd/test"
  done
ssh root@k8s-db01 "gluster volume create test-gv replica 2 k8s-db01:/home/glusterd/test k8s-db02:/home/glusterd/test force"
```

预期输出

``` bash
volume create: test-gv: success: please start the volume to access data
```

### 激活卷

``` bash
ssh root@k8s-db01 "gluster volume start test-gv"
```

预期输出

``` bash
volume start: test-gv: success
```

### 挂载

在k8s-node01节点挂载卷,并写入数据测试

``` bash
ssh root@k8s-node01 "mount -t glusterfs k8s-db01:/test-gv /mnt"
ssh root@k8s-node01 "echo volume > /mnt/test.txt"
```

确认下文件分别被写入到k8s-db01 和k8s-db02 

``` bash
cd /opt/k8s/work
export NODE_NAMES=(k8s-db01 k8s-db02)
for node_name in ${NODE_NAMES[@]}
  do
    echo ">>> ${node_name}"
    ssh root@${node_name} "ls -lh /home/glusterd/test"
  done
```

预期输出

``` bash
>>> k8s-db01
total 4.0K
-rw-r--r-- 2 root root 7 Mar 22 18:34 test.txt
>>> k8s-db02
total 4.0K
-rw-r--r-- 2 root root 7 Mar 22 18:34 test.txt
```

### 删除测试的卷

``` bash
ssh root@k8s-node01 "umount /mnt"
```

``` bash
[root@k8s-db01 ~]# gluster volume stop test-gv
Stopping volume will make its data inaccessible. Do you want to continue? (y/n) y
volume stop: test-gv: success
[root@k8s-db01 ~]# gluster volume delete test-gv
Deleting volume will erase all information about the volume. Do you want to continue? (y/n) y
volume delete: test-gv: success
```

## heketi配置

###  简介

Heketi提供了一个RESTful管理界面，可以用来管理GlusterFS卷的生命周期。 通过Heketi，就可以像使用OpenStack Manila，Kubernetes和OpenShift一样申请可以动态配置GlusterFS卷。Heketi会动态在集群内选择bricks构建所需的volumes，这样以确保数据的副本会分散到集群不同的故障域内。同时Heketi还支持任意数量的ClusterFS集群，以保证接入的云服务器不局限于单个GlusterFS集群。

[heketi项目地址](https://github.com/heketi/heketi)

### 安装 Heketi Client

在master节点上安装Heketi Client， 并解压到/opt/k8s/bin 目录下

``` bash
cd /opt/k8s/work
wget https://github.com/heketi/heketi/releases/download/v8.0.0/heketi-client-v8.0.0.linux.amd64.tar.gz
tar zxvf heketi-client-v8.0.0.linux.amd64.tar.gz
source /opt/k8s/bin/environment.sh 
for master_ip in ${MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp  heketi-client/bin/heketi-cli root@${master_ip}:/opt/k8s/bin/heketi-cli     
  done
```

预期输出

``` bash
>>> 10.50.41.41
heketi-cli       100%   43MB  43.5MB/s   00:01                      
>>> 10.50.41.42
heketi-cli       100%   43MB  43.5MB/s   00:01
```

注意： The client can be installed on any node/system in your network that has access to the Kubernetes cluster and GlusterFS cluster.

### 在集群中创建 Heketi secret

私钥在 **k8s-master02** 节点的 /root/.ssh/id_rsa

``` bash
$ kubectl create secret generic ssh-key-secret --from-file=private_key=/root/.ssh/id_rsa -n kube-system

secret/ssh-key-secret created
```

[挂载ssh私钥参考](https://kubernetes.io/zh/docs/concepts/configuration/secret/#%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B-%E5%8C%85%E5%90%AB-ssh-%E5%AF%86%E9%92%A5%E7%9A%84-pod)

### 获取和编辑 Heketi 配置文件以设置 SSH executor

heketi.json文件用于在初始容器启动时配置Heketi。 有多种方法可以创建并让Heketi正确访问和加载此文件，对于此示例，我们只是从文件中创建一个可以轻松传递到群集中的configmap。 或者，您可以简单地将heketi.json文件复制到每个节点并放入/usr/share/heketi并创建hostPath卷，甚至存储在另一个共享点（GlusterFS卷或NFS等等）。 此仓库包含样本heketi.json文件，您也可以从Heketi 仓库中获取最新文件。

``` bash
cd /opt/k8s/work
cat > heketi.json <<EOF
{
    "_port_comment": "Heketi Server Port Number",
    "port" : "8080",

    "_use_auth": "Enable JWT authorization. Please enable for deployment",
    "use_auth" : true,

    "_jwt" : "Private keys for access",
    "jwt" : {
        "_admin" : "Admin has access to all APIs",
        "admin" : {
            "key" : "glusterfs"
        },
        "_user": "User only has access to /volumes endpoint",
        "user": {
           "key": "glusterfs"
       }
    },


    "_glusterfs_comment": "GlusterFS Configuration",
    "glusterfs" : {
        "_executor_comment": "Execute plugin. Possible choices: mock, kubernetes, ssh",
        "executor" : "ssh",
        "sshexec" : {
            "rebalance_on_expansion": true,
            "keyfile" : "/etc/heketi/private_key",
            "port" : "22",
            "user" : "root",
            "fstab": "/etc/fstab"
        }

    }

}
EOF
```

在master节点上创建一个configmap

``` bash
cd /opt/k8s/work
kubectl create configmap heketi-config --from-file=/opt/k8s/work/heketi.json -n kube-system
```

预期输出：

``` bash
configmap/heketi-config created
```

### 创建专用的 GlusterFS 集群 endpoints 和 service

endpoints 和 service 用于访问GlusterFS集群以建立挂载。 这些在将来的步骤中用于我们的ReplicationController / Container，也可以重用于需要访问GlusterFS集群的任何其他pod。

``` bash
cd /opt/k8s/work
cat > glusterfs-endpoints.yaml <<EOF
apiVersion: v1
kind: Endpoints
metadata:
 name: glusterfs-cluster
 namespace: kube-system
subsets:
 - addresses:
   - ip: 10.50.41.61
   ports:
   - port: 1
     protocol: TCP
 - addresses:
   - ip: 10.50.41.62
   ports:
   - port: 1
     protocol: TCP
EOF
kubectl apply -f glusterfs-endpoints.yaml
```
预期输出：

``` bash
endpoints/glusterfs-cluster created
```

``` bash
cd /opt/k8s/work
cat > gluster-service.yaml <<EOF
kind: Service
apiVersion: v1
metadata:
  name: glusterfs-cluster
  namespace: kube-system
spec:
  ports:
  - port: 1
EOF
kubect apply -f gluster-service.yaml
```

预期输出：

``` bash
service/glusterfs-cluster created
```

###  创建 Heketi deployment 文件

``` bash
cd /opt/k8s/work
cat > heketi-deployment.yaml <<EOF
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: heketi-deployment
  namespace: kube-system
  labels:
    app: heketi
  annotations:
    description: Defines how to deploy Heketi
spec:
  replicas: 2
  template:
    metadata:
      name: heketi
      labels:
        app: heketi
    spec:
      containers:
      - name: heketi
        image: heketi/heketi:8
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        readinessProbe:
          timeoutSeconds: 3
          initialDelaySeconds: 3
          httpGet:
            path: "/hello"
            port: 8080
        livenessProbe:
          timeoutSeconds: 3
          initialDelaySeconds: 30
          httpGet:
            path: "/hello"
            port: 8080
        volumeMounts:
        - name: keys
          mountPath: /etc/heketi/private_key
          subPath:  private_key
        - name: config
          mountPath: /etc/heketi/heketi.json
          subPath: heketi.json
        - name: db
          mountPath: /var/lib/heketi
      volumes:
        - name: keys
          secret:
            secretName: ssh-key-secret
        - name: config
          configMap:
            name: heketi-config
        - name: db
          glusterfs:
            endpoints: glusterfs-cluster
            path: heketi
EOF
kubectl apply -f heketi-deployment.yaml
```

预期输出：

``` bash
deployment.extensions/heketi-deployment created
```

设置Heketi数据库卷和路径以存储heketi.db。 请注意，我们在实际的GlusterFS集群上使用了一个卷来存储此heketi.db（卷应该大于40Gb）。

heketi 卷要存在,手动创建卷

创建目录命令在k8s-db01 和 k8s-db02 节点上

创建卷命令在其中一个节点执行即可


```  bash
$ mkdir /opt/heketi
$ gluster volume create heketi replica 2 k8s-db01:/opt/heketi k8s-db02:/opt/heketi force
volume create: heketi: success: please start the volume to access data
$ gluster volume start heketi
volume start: heketi: success
$ gluster volume status heketi
Status of volume: heketi
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick k8s-db01:/opt/heketi                  49152     0          Y       6781 
Brick k8s-db02:/opt/heketi                  49152     0          Y       6727 
Self-heal Daemon on localhost               N/A       N/A        Y       6750 
Self-heal Daemon on k8s-db01                N/A       N/A        Y       6804 
 
Task Status of Volume heketi
------------------------------------------------------------------------------
There are no active volume tasks
```

### 创建 Heketi Service 文件

``` bash
cat > heketi-service.yaml <<EOF
kind: Service
apiVersion: v1
metadata: 
  name: heketi
  namespace: kube-system
  labels: 
    app: heketi   
  annotations: 
    description: "Exposes Heketi Service"
spec: 
  selector: 
    app: heketi
  ports: 
  - name: heketi
    port: 28080
    targetPort: 8080
EOF
kubectl apply -f heketi-service.yaml 
```

预期输出：

``` bash
service/heketi created
```

### 使用 ingress 转发对 heketi 的请求

编辑如下两个文件添加内容如下：

``` bash
vim /root/manifests/ingress-nginx/ingress-service.yaml
    - name: tcp-heketi
      port: 28080
      targetPort: 8080

vim /root/manifests/ingress-nginx/tcp-services-configmap.yaml
  28080: "kube-system/heketi:8080"

cd /root/manifests/ingress-nginx
kubectl apply -f tcp-services-configmap.yaml -f ingress-service.yaml
```

预期输出：

``` bash
configmap/tcp-services configured
service/ingress-nginx configured
```

### 测试连接 Heketi

``` bash
curl http://k8s-node01:28080/hello

或者
curl http://k8s-node02:28080/hello
```

预期输出：

``` bash
Hello from Heketi
```

### 确认 Heketi pod 在运行

``` bash
[root@k8s-master01 work]# kubectl get pod -n kube-system -l app=heketi
NAME                                 READY     STATUS    RESTARTS   AGE
heketi-deployment-84789cf8b9-gxd79   1/1       Running   0          14m
heketi-deployment-84789cf8b9-mjdlf   1/1       Running   0          14m

[root@k8s-master01 work]# kubectl logs -f heketi-deployment-84789cf8b9-gxd79 -n kube-system
Setting up heketi database
  File: /var/lib/heketi/heketi.db
  Size: 0               Blocks: 0          IO Block: 131072 regular empty file
Device: 89h/137d        Inode: 12919606720283354851  Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2019-03-23 09:00:47.873351041 +0000
Modify: 2019-03-23 09:00:47.873351041 +0000
Change: 2019-03-23 09:00:47.873351041 +0000
 Birth: -
Heketi v8.0.0-1-g082b556-release-8
[heketi] INFO 2019/03/23 09:59:05 Loaded ssh executor
[heketi] INFO 2019/03/23 09:59:05 GlusterFS Application Loaded
[heketi] INFO 2019/03/23 09:59:05 Started Node Health Cache Monitor
Authorization loaded
Listening on port 8080
[heketi] INFO 2019/03/23 09:59:15 Starting Node Health Status refresh
[heketi] INFO 2019/03/23 09:59:15 Cleaned 0 nodes from health cache
[heketi] INFO 2019/03/23 10:01:05 Starting Node Health Status refresh
[heketi] INFO 2019/03/23 10:01:05 Cleaned 0 nodes from health cache
```

### 为 Heketi Server 设置环境变量

``` bash
export HEKETI_CLI_SERVER=http://k8s-node01:28080
```

### 使用 Heketi with Gluster

拓扑文件用于告知Heketi有关GlusterFS存储环境以及将加载和管理的设备。

注意： Heketi目前仅限于管理raw设备，如果设备已经是Gluster卷，它将被跳过并被忽略。

### 创建并加载拓扑文件

确认目标主机上的raw设备

``` bash
cd /opt/k8s/work
export DB_IPS=(10.50.41.61 10.50.41.62)
for db_ip in ${DB_IPS[@]}
  do
    echo ">>> ${db_ip}"
    ssh root@${db_ip} lsblk   
  done
```

命令输出：

``` bash
>>> 10.50.41.61
NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0                     2:0    1    4K  0 disk 
sda                     8:0    0  100G  0 disk 
├─sda1                  8:1    0  500M  0 part /boot
└─sda2                  8:2    0 99.5G  0 part 
  └─centos_cent7-root 253:0    0 99.5G  0 lvm  /
sdb                     8:16   0    5T  0 disk 
├─sdb1                  8:17   0  1.4T  0 part 
└─sdb2                  8:18   0  1.4T  0 part 
sr0                    11:0    1 1024M  0 rom  
>>> 10.50.41.62
NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0                     2:0    1    4K  0 disk 
sda                     8:0    0  100G  0 disk 
├─sda1                  8:1    0  500M  0 part /boot
└─sda2                  8:2    0 99.5G  0 part 
  └─centos_cent7-root 253:0    0 99.5G  0 lvm  /
sdb                     8:16   0    5T  0 disk 
├─sdb1                  8:17   0  1.4T  0 part 
└─sdb2                  8:18   0  1.4T  0 part 
sr0                    11:0    1 1024M  0 rom 
```

从上面可以看出 可用的设备是/dev/sdb1 和 /dev/sdb2

``` bash
cd /opt/k8s/work
cat > topology.json <<EOF
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "k8s-db01"
              ],
              "storage": [
                "10.50.41.61"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sdb1",
              "destroydata": false
            },
            {
              "name": "/dev/sdb2",
              "destroydata": false
            }           
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "k8s-db02"
              ],
              "storage": [
                "10.50.41.62"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sdb1",
              "destroydata": false
            },          
            {
              "name": "/dev/sdb2",
              "destroydata": false
            }
          ]
        }   
      ]
    }
  ]
}
EOF
```

注意：上述json文件中的manage元素应该是GlusterFS节点的实际主机名，而storage元素应该是glusterfs-endpoints中存储节点的实际ip地址。

### 使用heketi-cli，运行以下命令以加载环境的拓扑

由于我们开启了heketi认证，所以每次执行heketi-cli操作时，都需要带上一堆的认证字段，比较麻烦，我在这里创建一个别名来避免相关操作：

``` bash
$ alias heketi-cli='heketi-cli --server "http://k8s-node01:28080" --user "admin" --secret "glusterfs"'
$ heketi-cli topology load --json=topology.json
```

上面的命令报错：

``` bash
$ heketi-cli topology load --json=topology.json                                                        
Creating cluster ... ID: d14e6a3946ce48d1cde19f49a5bb2512
        Allowing file volumes on cluster.
        Allowing block volumes on cluster.
        Creating node k8s-db01 ... Unable to create node: Cluster id does not exist
        Creating node k8s-db02 ... Unable to create node: New Node doesn't have glusterd running
```

查看Pod的日志发现是不能解析k8s-db02的IP地址

``` bash
[cmdexec] INFO 2019/03/23 10:48:19 Check Glusterd service status in node k8s-db02
[cmdexec] WARNING 2019/03/23 10:48:19 Failed to create SSH connection to k8s-db02:22: dial tcp: lookup k8s-db02 on 10.125.0.2:53: server misbehaving
[negroni] Completed 400 Bad Request in 5.131063ms
[cmdexec] ERROR 2019/03/23 10:48:19 /src/github.com/heketi/heketi/executors/cmdexec/peer.go:76: dial tcp: lookup k8s-db02 on 10.125.0.2:53: server misbehaving
[heketi] ERROR 2019/03/23 10:48:19 /src/github.com/heketi/heketi/apps/glusterfs/app_node.go:107: dial tcp: lookup k8s-db02 on 10.125.0.2:53: server misbehaving
[heketi] ERROR 2019/03/23 10:48:19 /src/github.com/heketi/heketi/apps/glusterfs/app_node.go:108: New Node doesn't have glusterd running
```

解决方法：

+ 第一： heketi-deployment.yaml 文件中添加 ”hostNetwork: true“  并重新执行该yaml文件
+ 第二： coredns中添加a记录，使pod可以解析该IP 

选择第一种方式，因此上面的nginx转发可以不需要了

重新加载拓扑

``` bash
alias heketi-cli='heketi-cli --server "http://k8s-node01:8080" --user "admin" --secret "glusterfs"'
heketi-cli topology load --json=topology.json
```

输出如下：

``` bash
Creating cluster ... ID: 77e6b8439bbd4ea8a7c75bfc308083a3
        Allowing file volumes on cluster.
        Allowing block volumes on cluster.
        Creating node k8s-db01 ... ID: 44ea578b30528b4b4055f03c862c0f63
                Adding device /dev/sdb1 ... OK
                Adding device /dev/sdb2 ... OK
        Creating node k8s-db02 ... ID: 2ebefadc6d84c1700ba09544bb9eede3
                Adding device /dev/sdb1 ... OK
                Adding device /dev/sdb2 ... OK
```

### 创建一个 Gluster 卷确认 Heketi 正常工作

``` bash
heketi-cli volume create --size=20 --replica=2

Name: vol_055c6d1934bf8f09f05c75552e33c54b
Size: 20
Volume Id: 055c6d1934bf8f09f05c75552e33c54b
Cluster Id: 77e6b8439bbd4ea8a7c75bfc308083a3
Mount: 10.50.41.62:vol_055c6d1934bf8f09f05c75552e33c54b
Mount Options: backup-volfile-servers=10.50.41.61
Block: false
Free Size: 0
Reserved Size: 0
Block Hosting Restriction: (none)
Block Volumes: []
Durability Type: replicate
Distributed+Replica: 2
```

### 从其中一个 Gluster 节点查看卷信息

``` bash
cd /opt/k8s/work
ssh root@k8s-db01 'gluster volume list'
ssh root@k8s-db02 'gluster volume info vol_055c6d1934bf8f09f05c75552e33c54b'
```

输出如下：
``` bash
heketi
vol_055c6d1934bf8f09f05c75552e33c54b

 
Volume Name: vol_055c6d1934bf8f09f05c75552e33c54b
Type: Replicate
Volume ID: d891d3cf-fdd7-4426-a7a1-18de361ddd1b
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: 10.50.41.62:/var/lib/heketi/mounts/vg_d1c6ad71d49f31becd5b023c2ca578dc/brick_1e8e4103ad41ead3be8b0398cb12578b/brick
Brick2: 10.50.41.61:/var/lib/heketi/mounts/vg_e09ead39669fd3cd995e0b6e3d72050c/brick_fd89dabd5bfe52f29fc76e039b90d6d4/brick
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

### 动态卷供给.

#### 创建一个存储池

官方推荐将key使用secret保存


``` bash
cat > glusterfs-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
data:
  # base64 encoded password. E.g.: echo -n "mypassword" | base64
  key: Z2x1c3RlcmZz
type: kubernetes.io/glusterfs
```

``` bash
cat > glusterfs-storageclass.yaml <<EOF
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://k8s-node01:8080"
  clusterid: "77e6b8439bbd4ea8a7c75bfc308083a3"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "kube-system"
  secretName: "heketi-secret"  
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:2"
EOF
```
执行yaml文件

``` bash
$ kubectl apply -f glusterfs-secret.yaml 
secret/heketi-secret created

$ kubectl apply -f glusterfs-storageclass.yaml 
storageclass.storage.k8s.io/glusterfs created

$ kubectl get storageclasses.storage.k8s.io 
NAME        PROVISIONER               AGE
glusterfs   kubernetes.io/glusterfs   10s
```

### 创建一个pvc测试下

``` bash
cat > pvc-test.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
  annotations:
    volume.beta.kubernetes.io/storage-class: glusterfs
spec:
 accessModes:
  - ReadWriteMany
 resources:
   requests:
     storage: 20Gi
EOF
```

``` bash
$ kubectl apply -f pvc-test.yaml
  persistentvolumeclaim/pvc-test created

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM              STORAGECLASS   REASON    AGE
pvc-2feef9db-4d64-11e9-b6cd-005056aa711f   20Gi       RWX            Delete           Bound     default/pvc-test   glusterfs   
             2m
$ kubectl get pvc
NAME       STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound     pvc-2feef9db-4d64-11e9-b6cd-005056aa711f   20Gi       RWX            glusterfs      2m
```

### 创建一个 NGINX pod 使用这个 PVC

``` bash
cat > nginx-pvc.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gluster-pod1
  labels:
    name: gluster-pod1
spec:
  containers:
  - name: gluster-pod1
    image: gcrxio/nginx-slim:0.8
    ports:
    - name: web
      containerPort: 80
    securityContext:
      privileged: true
    volumeMounts:
    - name: gluster-vol1
      mountPath: /usr/share/nginx/html
  volumes:
  - name: gluster-vol1
    persistentVolumeClaim:
      claimName: pvc-test
EOF
```

``` bash
$ kubectl apply -f nginx-pvc.yaml 
pod/gluster-pod1 created

$ kubectl get pod  -o wide
NAME           READY     STATUS    RESTARTS   AGE       IP            NODE         NOMINATED NODE
gluster-pod1   1/1       Running   0          1m        10.124.56.4   k8s-node02   <none>
```

### 进入容器并创建一个index.html文件

``` bash
$ kubectl  exec -it gluster-pod1 /bin/bash
$ cd /usr/share/nginx/html/
$ echo 'Hello World from GlusterFS!!!' >  index.html
$ ls -lh
total 512
-rw-r--r-- 1 root 40000 30 Mar 23 12:19 index.html
$ exit
```

### 访问nginx

``` bash
curl http://10.124.56.4
Hello World from GlusterFS!!!
```
###  确认共享存储中的文件是两份

``` bash
[root@k8s-db02 log]# df -hT
文件系统                                                                               类型      容量  已用  可用 已用% 挂载点
/dev/mapper/centos_cent7-root                                                          xfs       100G  1.3G   99G    2% /
devtmpfs                                                                               devtmpfs  3.9G     0  3.9G    0% /dev
tmpfs                                                                                  tmpfs     3.9G     0  3.9G    0% /dev/shm
tmpfs                                                                                  tmpfs     3.9G  8.6M  3.9G    1% /run
tmpfs                                                                                  tmpfs     3.9G     0  3.9G    0% /sys/fs/cgroup
/dev/sda1                                                                              xfs       497M  136M  362M   28% /boot
/dev/mapper/vg_d1c6ad71d49f31becd5b023c2ca578dc-brick_1e8e4103ad41ead3be8b0398cb12578b xfs        20G   33M   20G    1% /var/lib/heketi/mounts/vg_d1c6ad71d49f31becd5b023c2ca578dc/brick_1e8e4103ad41ead3be8b0398cb12578b
/dev/mapper/vg_36735d2046d91669321ec7a44b64a96d-brick_44554779ae9f61385c9cf6851178d902 xfs        20G   34M   20G    1% /var/lib/heketi/mounts/vg_36735d2046d91669321ec7a44b64a96d/brick_44554779ae9f61385c9cf6851178d902
tmpfs                                                                                  tmpfs     782M     0  782M    0% /run/user/0
ls -lh /var/lib/heketi/mounts/vg_36735d2046d91669321ec7a44b64a96d/brick_44554779ae9f61385c9cf6851178d902/brick/

总用量 4.0K
-rw-r--r-- 2 root 40000 30 3月  23 20:19 index.html
```

可以看到我们刚创建的index.html文件

参考文档：

https://www.cnblogs.com/breezey/p/8849466.html

https://wiki.centos.org/SpecialInterestGroup/Storage/gluster-Quickstart

https://kubernetes.io/docs/concepts/storage/storage-classes/#glusterfs

https://github.com/gluster/gluster-kubernetes/blob/master/docs/examples/containerized_heketi_dedicated_gluster/README.md
