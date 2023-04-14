#### capi自定义使用captain安装节点-问题
1.kubeadm仅仅初始化自己  并向自己启动的第一个带有公网ip的machine 发送注册node请求 因此 不会存在网络问题(过程存在问题 多master不开启LB 只有一个公网ip不足以分配 否则需要使用LB作为其他master的注册node端点)
   ansible初始化 1.目标下发  2.自己的创建 及状态维护
2. capi 初始化顺序为启动一个master->启动多个work并启动多个master，一切的基础是第一个master，当第一个master没有ready(这里的判断标准是初始化为一个k8s节点 即可以向此节点注册node 同1) 不会创建后面的虚拟机
   ansible更倾向于所有虚拟机都创建好  因此需要解除capi对第一个master未创建好时 work的限制
   ansible 虚拟机 规格，预留资源
3.使用capi 组件部署 bootstrap部分 即capi-kubeadm-bootstrap-controller-manager部分 需要解除对此组件和下面crd的依赖（直线代表管理 虚线代表引用）
![Screenshot from 2022-06-14 10-16-58.png](https://cluster-api.sigs.k8s.io/images/worker-machines-resources.png)


4.使用capi 组件部署kubeadm-control-plane-controller 部分,对于集群master(etcd)的配置更新 均需要自己实现 即解除对此组件和下面crd的依赖 
![Screenshot from 2022-07-19 16-56-17.png](https://cluster-api.sigs.k8s.io/images/kubeadm-control-plane-machines-resources.png)


5.镜像的重制(ansible 等)

#### capi安装节点-优势
1.crd定义-更符合云原生
2.以删除一个节点并扩展回来为例(你意图删除一个虚拟机，假设虚拟机文件系统损坏) 直接删除machine crd即可，处理过程中控制器会去删除node并做drain等操作，然后自动创建并注册此节点。
3.无额外资源消耗 无ansible安装机等 轻量
4.错误输出 查询关键资源event即可

#### capi安装节点-劣势
1.删除整个集群资源感觉很麻烦 
2.需要自己完成自定义资源的安装

#### 问题 如何做到？
1.集群节点配置变更，每个节点上做某些修改，例如：修改一个文件，创建一个目录(难以做到 只负责集群创建)
2.集群稳定扩缩容，新扩容一批worker节点，删除某个worker节点(不支持指定名称的缩放-可以考虑先删除此machine 使其处于deleting状态 然后对控制器进行缩放 避免创建新的)
3.集群组件升级：k8s/containerd/etcd 及相关组件升级-（使用kubeadm config支持k8s,etcd组件升级  containerd属于镜像范畴）
4.集群网络方案切换 (不支持)
5.集群节点更换镜像以及分区(更换镜像支持 触发操作为copy machinetemplate配置 修改镜像 修改KubeadmControlPlane或者machinedeployment资源引用  分区更换 现在仅支持创建新的分区节点并删除旧的 work可以做到 控制平面对于区域的更新则更为复杂 首先控制平面目前是无法支持自定义分区的 支持的情况是OpenStackCluster可以从定义的分区数组中进行验证区域的可行性 并标记在此资源的status下，cluster资源会同步此资源的status,KubeadmControlPlane 调谐则在创建machine资源时从cluster status区域中选择当前machine最少的区域进行machine的创建)
6.一个集群多种系统架构(由于work节点是能够精确到单节点和az设置镜像 因此可以做到一个集群多架构 master来看目前不支持此操作 因为只引用了一个openstackmachinetemplate 需要额外代码实现)

controller plane 控制平面创建泳道图
1.cluster资源创建部分
![Screenshot from 2022-06-14 10-16-58.png](https://raw.githubusercontent.com/kubernetes-sigs/cluster-api/main/docs/proposals/images/controlplane/controlplane-init-1.png)
2.使用kubeadm创建后部分
![Screenshot from 2022-06-14 10-16-58.png](https://raw.githubusercontent.com/kubernetes-sigs/cluster-api/main/docs/proposals/images/controlplane/controlplane-init-2.png)
3.machine status update
![Screenshot from 2022-06-14 10-16-58.png](https://raw.githubusercontent.com/kubernetes-sigs/cluster-api/main/docs/proposals/images/controlplane/controlplane-init-3.png)
4.kubeadm update
![Screenshot from 2022-06-14 10-16-58.png](https://raw.githubusercontent.com/kubernetes-sigs/cluster-api/main/docs/proposals/images/controlplane/controlplane-init-4.png)

汇总
1.检测内置脚本成功与否.
想法 无 当集群部署完毕后 谁来触发更新？
resourceset 属于集群只缺少cni等 其他均已经部署完毕 

2.修改openstackmachine name (
controlplane负责master openstackmachine 的创建name 位置
/home/jianmei/go/src/cluster-api/controlplane/kubeadm/internal/controllers/helpers.go 169行
创建machine 并创建引用
machinedeployment 创建openstack machine的位置应该位于machineset代码中
/home/jianmei/go/src/cluster-api/internal/controllers/machineset/machineset_controller.go 396行
)
name可以指定 但如何对映射关系进行处理

machinedeployment资源引用OpenStackMachineTemplate资源和KubeadmConfigTemplate资源 machinedeployment资源下创建machinset  此资源创建OpenStackMachineTemplate对应的openstackmachine子和KubeadmConfigTemplate对应的kubeadmconfigs.bootstrap.cluster.x-k8s.io资源 并创建machine.
因此 若从cr字段上指定 则由于machinedeploument不确定副本数目性 不能在此指定 需要在machine资源中进行指定 不使用machinedeployment?直接使用代码创建machine资源？ 

3.修改pvc 执行删除 集群内部LB删除  集群内部pvc 删除问题（删除集群时 添加自定义代码 对特定的描述的LB和cinder卷进行回收   etcd盘和自定义盘 应在machine删除时被回收 在machine创建时被创建 定义在openstackTemplate 的spec中 实际引用openstackmachinemachine spec,可以定义类似于
```
RootVolume     *RootVolume       `json:"rootVolume,omitempty"`
```
定义
```
EtcdVolume     *RootVolume       `json:"etcdVolume,omitempty"`
CustomeVolume  []*RootVolume      `json:"etcdVolume,omitempty"`
```
）
在/home/jianmei/go/src/cluster-api-provider-openstack/pkg/cloud/services/compute/instance.go 191行修改其中的卷创建流程 对etcdvolume和CustomeVolume 进行创建

在/home/jianmei/go/src/cluster-api-provider-openstack/pkg/cloud/services/compute/instance.go 244行进行追加etcdvolume和CustomeVolume 挂载创建虚拟机

修改master cloud init文件
/home/jianmei/go/src/cluster-api/bootstrap/kubeadm/internal/cloudinit/controlplane_init.go
/home/jianmei/go/src/cluster-api/bootstrap/kubeadm/internal/cloudinit/controlplane_join.go 
需对挂载的盘进行初始化  并挂载到文件系统(etcd数据目录)

社区https://github-com.translate.goog/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20200423-etcd-data-disk.md?_x_tr_sl=auto&_x_tr_tl=zh-CN&_x_tr_hl=en
4.租户与权限 目前支持的凭据为
目前看支持直接凭据如下
```
CAPO_AUTH_URL=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.auth_url)
CAPO_USERNAME=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.username)
CAPO_PASSWORD=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.password)
CAPO_REGION=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.region_name)
CAPO_PROJECT_ID=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.project_id)
CAPO_PROJECT_NAME=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.project_name)
CAPO_DOMAIN_NAME=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.user_domain_name)
CAPO_APPLICATION_CREDENTIAL_NAME=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.application_credential_name)
CAPO_APPLICATION_CREDENTIAL_ID=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.application_credential_id)
CAPO_APPLICATION_CREDENTIAL_SECRET=$(echo "$CAPO_OPENSTACK_CLOUD_YAML_CONTENT" | yqNavigating - clouds.${CAPO_CLOUD}.auth.application_credential_secret)
```
对trust 用户应该不支持 不知道是否可以work around
5.LB的创建
目前现有的集群eks是不依赖于LB 为每个master分配一个公网ip ，当不使用LB时 会造成多master下申请同一个fip并绑定 导致错误的创建machine 最好检查多master后 直接对其他master申请未使用的fip并绑定
/home/jianmei/go/src/cluster-api-provider-openstack/controllers/openstackcluster_controller.go 510行
/home/jianmei/go/src/cluster-api-provider-openstack/controllers/openstackmachine_controller.go 362行
6.打标签
没有想法
7.控制面指定多区域
控制面创建machine指定的区域代码位于/home/jianmei/go/src/cluster-api/controlplane/kubeadm/internal/controllers/controller.go 360  365行
修改选择创建machine资源的az ,需要输入master节点和对应az的对应关系？ 如何读取？
8.节点网络 
当openstackcluster设置NodeCIDR 会自动取创建对应的网络 子网 路由器
否则 使用现成的网络 网络目前可以设置过滤器过滤特定网络 子网也是如此 
master处于同一个openstacktemplate定义的网络和子网下  
work节点因为可以有多个openstacktemplate 则可以分开配置
安全组首先内置默认安装组 不存在会创建
work默认安全组如下
```
出口 	IPv4 	任何	任何 	网段： 0.0.0.0/0 	
入口 	IPv4 	tcp	179 	安全组： k8s-cluster-default-test-secgroup-worker 	
入口 	IPv4 	tcp	10250 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
入口 	IPv4 	tcp	179 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
入口 	IPv4 	tcp	10250 	安全组： k8s-cluster-default-test-secgroup-worker 	
入口 	IPv4 	4	任何 	安全组： k8s-cluster-default-test-secgroup-worker 	
入口 	IPv4 	4	任何 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
入口 	IPv4 	tcp	30000~32767 	网段： 0.0.0.0/0 	
出口 	IPv6 	任何	任何 	网段： ::/0 	
```
master默认安全组
```
 入口 	IPv4 	tcp	179 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
入口 	IPv4 	tcp	2379~2380 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
出口 	IPv4 	任何	任何 	网段： 0.0.0.0/0 	
入口 	IPv4 	4	任何 	安全组： k8s-cluster-default-test-secgroup-worker 	
入口 	IPv4 	tcp	6443 	网段： 0.0.0.0/0 	
入口 	IPv4 	tcp	10250 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
入口 	IPv4 	4	任何 	安全组： k8s-cluster-default-test-secgroup-controlplane 	
入口 	IPv4 	tcp	10250 	安全组： k8s-cluster-default-test-secgroup-worker 	
出口 	IPv6 	任何	任何 	网段： ::/0 	
入口 	IPv4 	tcp	179 	安全组： k8s-cluster-default-test-secgroup-worker 	
```
可以自定义添加自己openstack中写好的安全组
9 指定work节点的删除 并不创建新的节点 依赖副本数的做法
- RandomMachineSetDeletePolicy MachineSetDeletePolicy = "Random"此删除策略下
使用DeleteMachineAnnotation = "cluster.x-k8s.io/delete-machine" 此标记标记machine 在scale down副本数时会优先考虑在此节点和不健康的机器随机选择.
- NewestMachineSetDeletePolicy MachineSetDeletePolicy = "Newest" 此删除策略
使用DeleteMachineAnnotation = "cluster.x-k8s.io/delete-machine" 此标记标记machine 则在scale down副本数时会优先考虑在此节点和不健康的机器选择创建时间更加后面的机器
- OldestMachineSetDeletePolicy MachineSetDeletePolicy = "Oldest" 此删除策略
使用DeleteMachineAnnotation = "cluster.x-k8s.io/delete-machine" 此标记标记machine 则在scale down副本数时会优先考虑在此节点和不健康的机器选择创建时间更加靠前的机器

machine不健康的判断依据为Status.FailureReason or Status.FailureMessage 没有设置为空或者
NodeHealthy type of Status.Conditions 不为 true) 前面两个条件基本是创建虚拟机失败与否 最后一个条件的判断标准是work集群kubectl get node 列表中有无此节点  因此当节点不健康 直接收缩 就可以剔除不健康节点
至于健康节点的删除 直接依赖于cluster.x-k8s.io/delete-machine:true 此注解即可



路径：
输入：虚拟机(ip) +ansible









