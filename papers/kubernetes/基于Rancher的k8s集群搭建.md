#基于Rancher的k8s集群的搭建

-----

## 前置条件
1. 操作系统：CentOS 7
2. 至少3台机器，一台作为rancher server(假定为rancher-server)，另外2台作为k8s集群的节点机器（假定为node1和node2）。

## 清理机器

如果机器之前安装过rancher，则参考以下文章对机器进行清理：https://rancher.com/docs/rancher/v2.x/en/cluster-admin/cleaning-cluster-nodes/

1. 清理docker
```bash
sudo docker rm -f $(docker ps -qa)
sudo docker rmi -f $(docker images -q)
sudo docker volume rm $(docker volume ls -q)
```
2. 清理kubelet
```bash
for mount in $(mount | grep tmpfs | grep '/var/lib/kubelet' | awk '{ print $3 }') /var/lib/kubelet /var/lib/rancher; do sudo umount $mount; done
```
3. 清理文件以及文件夹
```bash
sudo rm -rf /etc/ceph \
       /etc/cni \
       /etc/kubernetes \
       /opt/cni \
       /opt/rke \
       /run/secrets/kubernetes.io \
       /run/calico \
       /run/flannel \
       /var/lib/calico \
       /var/lib/etcd \
       /var/lib/cni \
       /var/lib/kubelet \
       /var/lib/rancher/rke/log \
       /var/log/containers \
       /var/log/pods \
       /var/run/calico
```
4. 清理网络和iptables：重启即可。
```bash
sudo reboot
```

## 安装Rancher Server
登录rancher-server，执行以下操作：
1. 安装docker
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce
```
1. 安装rancher server
```bash
sudo docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /root/certs:/container/certs \
  -v /opt/rancher:/var/lib/rancher \
  -e SSL_CERT_DIR="/container/certs" \
  rancher/rancher:v2.2
```

## 新建k8s集群

1. 安装docker
```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce
```
2. 设置机器名称
```bash
sudo hostnamectl set-hostname node-[x]
```
3. 禁用防火墙
```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```
4. 刷新iptables
```bash
sudo iptables -tnat --flush
sudo iptables --flush
```
3. 在浏览器中访问 https://[rancher-server的ip],设置管理员密码后，登录系统后，选择建立集群，输入集群名称，选择网络插件类型；加入集群的第一台机器的角色选择etcd，control plane和worker，然后在要加入集群的节点执行 网页中显示的命令。
4. 依次对其它要加入集群的节点执行以上操作。

## 安装k8s中持久化存储的插件openEBS
参考：
* https://docs.openebs.io/docs/next/prerequisites.html
* https://docs.openebs.io/docs/next/prerequisites.html#rancher

在所有加入集群的节点中执行以下操作：
1. 启用iscsi的功能
```bash
sudo yum install iscsi-initiator-utils -y
sudo systemctl enable iscsid
sudo systemctl start iscsid
sudo modprobe iscsi_tcp
sudo echo iscsi_tcp > /etc/modules-load.d/iscsi_tcp.conf
```
2. 登录Rancher的管理界面，选择指定集群的菜单中，选择修改集群，在yaml中添加以下内容：
```yaml
services:
    kubelet: 
      extra_binds: 
        - "/etc/iscsi:/etc/iscsi"
        - "/sbin/iscsiadm:/sbin/iscsiadm"
        - "/var/lib/iscsi:/var/lib/iscsi"
        - "/lib/modules"
```
3. 安装openebs
``` 
kubectl apply -f https://openebs.github.io/charts/openebs-operator-1.0.0.yaml
``` 
4. 在k8s中建立cstor-pool, kubectl apply -f openebs-cstor-pool-config.yaml
```yaml
# openebs-cstor-pool-config.yaml
#Use the following YAMLs to create a cStor Storage Pool.
# and associated storage class.
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-pool
  annotations:
    cas.openebs.io/config: |
      #- name: PoolResourceRequests
        #value: |-
            #memory: 2Gi
      - name: PoolResourceLimits
        value: |-
            memory: 2Gi
spec:
  name: cstor-pool1
  type: disk
  maxPools: 3
  poolSpec:
    poolType: striped
  # NOTE - Appropriate disks need to be fetched using `kubectl get disks`
  #
  # `Disk` is a custom resource supported by OpenEBS with `node-disk-manager`
  # as the disk operator
# Replace the following with actual disk CRs from your cluster `kubectl get disks`
# Uncomment the below lines after updating the actual disk names.
  disks:
    diskList:
# Replace the following with actual disk CRs from your cluster from `kubectl get disks`
      - disk-a712038824d3bcd1726b1f0e73320ff6
      - disk-ab303715036b01273493b005844a35a7
      - disk-ef86fb3a5392423a92aabca6067be15b
```
5. 在k8s中建立storeClass, kubectl apply -f ./openebs-cstor-deploy.yaml
```yaml
# openebs-cstor-deploy.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-cstor-deploy
  annotations:
    openebs.io/cas-type: cstor
    cas.openebs.io/config: |
      - name: StoragePoolClaim
        value: "cstor-pool"
      - name: ReplicaCount
        value: "3"
provisioner: openebs.io/provisioner-iscsi
reclaimPolicy: Delete
```
6.查看openebs状态

``` 
kubectl get pods -n openebs -o wide
kubectl get svc -n openebs
kubectl get crd
``` 


