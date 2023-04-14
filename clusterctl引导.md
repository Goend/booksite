


删除apiserver --feature-gates ServerSideApply=True

写入host
```
140.82.114.25                 alive.github.com
140.82.113.26                 live.github.com
185.199.108.154               github.githubassets.com
140.82.112.21                 central.github.com
185.199.108.133               desktop.githubusercontent.com
185.199.108.153               assets-cdn.github.com
185.199.108.133               camo.githubusercontent.com
185.199.108.133               github.map.fastly.net
146.75.77.194                 github.global.ssl.fastly.net
140.82.113.3                  gist.github.com
185.199.108.153               github.io
140.82.114.3                  github.com
#20.205.243.166                github.com
192.0.66.2                    github.blog
140.82.114.6                  api.github.com
185.199.108.133               raw.githubusercontent.com
185.199.108.133               user-images.githubusercontent.com
185.199.108.133               favicons.githubusercontent.com
185.199.108.133               avatars5.githubusercontent.com
185.199.108.133               avatars4.githubusercontent.com
185.199.108.133               avatars3.githubusercontent.com
185.199.108.133               avatars2.githubusercontent.com
185.199.108.133               avatars1.githubusercontent.com
185.199.108.133               avatars0.githubusercontent.com
185.199.108.133               avatars.githubusercontent.com
140.82.113.10                 codeload.github.com
52.216.137.84                 github-cloud.s3.amazonaws.com
52.217.74.92                  github-com.s3.amazonaws.com
54.231.169.129                github-production-release-asset-2e65be.s3.amazonaws.com
52.217.136.185                github-production-user-asset-6210df.s3.amazonaws.com
52.216.169.115                github-production-repository-file-5c1aeb.s3.amazonaws.com
185.199.108.153               githubstatus.com
64.71.144.211                 github.community
23.100.27.125                 github.dev
140.82.112.22                 collector.github.com
13.107.43.16                  pipelines.actions.githubusercontent.com
185.199.108.133               media.githubusercontent.com
185.199.108.133               cloud.githubusercontent.com
#185.199.110.133               objects.githubusercontent.com
185.199.108.133               objects.githubusercontent.com
```
下载clusterctl二进制
```
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.1.4/clusterctl-linux-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl

```

下载镜像文件 主要包括四个
2.2 下载 ClusterAPI 组件的 yaml 文件

ClusterAPI 组件的默认仓库地址，可以通过 clusterctl config repositories 命令查看，下面列出的是此次实验需要用到的文件：

root@cluster-api:~# clusterctl config repositories
NAME           TYPE                     URL                                                                                          FILE
cluster-api    CoreProvider             https://github.com/kubernetes-sigs/cluster-api/releases/latest/                              core-components.yaml
kubeadm        BootstrapProvider        https://github.com/kubernetes-sigs/cluster-api/releases/latest/                              bootstrap-components.yaml
kubeadm        ControlPlaneProvider     https://github.com/kubernetes-sigs/cluster-api/releases/latest/                              control-plane-components.yaml
openstack      InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-openstack/releases/latest/           infrastructure-components.yaml

其实这些 yaml 文件都可以通过 clusterctl generate provider 命令生成，但是我选择直接手动下载，因为我们还需要相关的元数据文件。
通过https://github.com/kubernetes-sigs/cluster-api/releases/latest/下载
`core-components.yaml、bootstrap-components.yaml、control-plane-components.yaml、metadata.yaml`

通过https://github.com/kubernetes-sigs/cluster-api-provider-openstack/releases/latest/下载
`infrastructure-components.yaml、cluster-template.yaml 和 metadata.yaml。`
通过https://github.com/cert-manager/cert-manager/releases下载
`cert-manager.yaml`

2.3 修改 image 下载地址(后面部分为自己的)

sed -i s#"k8s.gcr.io/cluster-api"#"k8s-gcr.panbuhei.online/cluster-api"#g core-components.yaml 
sed -i s#"k8s.gcr.io/cluster-api"#"k8s-gcr.panbuhei.online/cluster-api"#g bootstrap-components.yaml 
sed -i s#"k8s.gcr.io/cluster-api"#"k8s-gcr.panbuhei.online/cluster-api"#g control-plane-components.yaml 

sed -i s#"k8s.gcr.io/capi-openstack"#"k8s-gcr.panbuhei.online/capi-openstack"#g infrastructure-components.yaml

PS： 还需要关注 cluster-metadata.yaml 中 nodeCidr 和 podCidr 的网络地址段，以防冲突。

    nodeCidr 默认网段为：10.6.0.0/24。
  podCidr 默认网段为：192.168.0.0/16。

·
3、修改完成后，配置 Overrides 层

Overrides 层的默认路径为 $HOME/.cluster-api/overrides。可以在 clusterctl 配置文件(默认 $HOME/.cluster-api/clusterctl.yaml)中通过 overridesFolder: <PATH> 来指定。

此目录结构应遵循 <providerType-providerName>/<version>/<fileName> 模板。如下所示：

root@cluster-api:~# tree ~/.cluster-api/overrides/
/root/.cluster-api/overrides/
├── bootstrap-kubeadm
│   └── v1.1.4
│       ├── bootstrap-components.yaml
│       └── metadata.yaml
├── cert-manager
│   └── v1.7.2
│       └── cert-manager.yaml
├── cluster-api
│   └── v1.1.4
│       ├── core-components.yaml
│       └── metadata.yaml
├── control-plane-kubeadm
│   └── v1.1.4
│       ├── control-plane-components.yaml
│       └── metadata.yaml
└── infrastructure-openstack
    └── v0.6.3
        ├── cluster-template.yaml
        ├── infrastructure-components.yaml
        └── metadata.yaml

编辑或修改 clusterctl 配置文件

```
cat > $HOME/.cluster-api/clusterctl.yaml << "EOF"
providers:
  - name: "cluster-api"
    url: "/root/.cluster-api/overrides/cluster-api/latest/core-components.yaml"
    type: "CoreProvider"
  - name: "kubeadm"
    url: "/root/.cluster-api/overrides/bootstrap-kubeadm/latest/bootstrap-components.yaml"
    type: "BootstrapProvider"
  - name: "kubeadm"
    url: "/root/.cluster-api/overrides/control-plane-kubeadm/latest/control-plane-components.yaml"
    type: "ControlPlaneProvider"
  - name: "openstack"
    url: "/root/.cluster-api/overrides/infrastructure-openstack/latest/infrastructure-components.yaml"
    type: "InfrastructureProvider"
cert-manager:
  url: "/root/.cluster-api/overrides/cert-manager/latest/cert-manager.yaml"
  version: "v1.7.2"
EOF
```

一旦指定了这些 overrides，clusterctl 将使用它们而不是从默认或指定的提供程序中获取值。再次执行 clusterctl config repositories 命令就可以看出。

编译符合capi的openstack镜像
```
git clone https://github.com/kubernetes-sigs/image-builder.git
```
按照此文档操作进行
https://image--builder-sigs-k8s-io.translate.goog/capi/providers/openstack.html?_x_tr_sl=auto&_x_tr_tl=zh-CN&_x_tr_hl=en
当使用root权限操作 可以跳过kvm赋权阶段


images/capi/packer/config 中包含几个 JSON 文件，它们定义了 image 的默认配置。如果需要自定义，不需要直接修改它们的值，最好是创建一个新 JSON 文件，并通过 PACKER_VAR_FILES 环境变量指定它。在这个文件中设置的变量将覆盖之前的任何值。

PS： 可以通过 PACKER_VAR_FILES 传递多个文件，最后一个文件优先于任何其他文件。

cd image-builder/images/capi/
```
cat << 'EOF' > packer/config/vars.json
{
  "kubernetes_container_registry": "registry.aliyuncs.com/google_containers",
  "pause_image": "registry.aliyuncs.com/google_containers/pause:3.5",
  "kubernetes_deb_gpg_key": "https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg",
  "kubernetes_deb_repo": "\"https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial\"",
  "kubernetes_deb_version": "1.22.8-00",
  "kubernetes_series": "v1.22",
  "kubernetes_semver": "v1.22.8",
  "containerd_url": "http://10.0.30.29/K8sFileDownload/github.com/containerd/containerd/releases/download/v1.6.2/cri-containerd-cni-1.6.2-linux-amd64.tar.gz",
  "containerd_sha256": "91f1087d556ecfb1f148743c8ee78213cd19e07c22787dae07fe6b9314bec121",
  "containerd_version": "1.6.2",
  "crictl_url": "http://10.0.30.29/K8sFileDownload/github.com/kubernetes-sigs/cri-tools/releases/download/v1.23.0/crictl-v1.23.0-linux-amd64.tar.gz",
  "crictl_sha256": "b754f83c80acdc75f93aba191ff269da6be45d0fc2d3f4079704e7d1424f1ca8",
  "crictl_version": "1.23.0",
  "goss_download_path": "http://10.0.30.29/K8sFileDownload/github.com/aelsabbahy/goss/releases/download/v0.3.16/goss-linux-amd64",
  "goss_version": "0.3.16"
}
EOF
```
registry.aliyuncs.com/google_containers
这些自定义变量主要是替换了软件包的默认下载地址，其中 containerd、crictl 和 goss 的下载地址默认是 GitHub，由于网络的限制，我们需要提前将其手动下载，然后通过自己的文件服务发布程序(如上的10.0.30.29)进行发布，这样可以加速 kubernetes 集群安装的速度，并能提高成功率。

构建image
```
cd image-builder/images/capi/
PACKER_VAR_FILES=packer/config/vars.json make build-qemu-ubuntu-2004

cd images/capi/output/BUILD_NAME+kube-KUBERNETES_VERSION 即可找到镜像

```
openstack导入镜像
```
glance image-create --name "ubuntu-2004-kube-xx" \
--file  {file}\
--disk-format qcow2 --container-format bare --visibility public
```


初始化控制面
```
clusterctl init --infrastructure openstack --target-namespace capi-system(这里可以选择组件--core,--bootstrap,
--control-plane。默认全装)
```


配置文件
cat << "EOF" > clouds.yaml
clouds:
  test:
    identity_api_version: 3
    auth:
      auth_url: http://keystone.openstack.svc.cluster.local/v3
      project_domain_name: Default
      user_domain_name: Default
      project_name: admin
      username: admin
      password: Admin@ES20!8
    region_name: RegionOne
EOF

下载脚本
https://github.com/kubernetes-sigs/cluster-api-provider-openstack/blob/main/templates/env.rc
重命名为env.rc
wget https://github.com/mikefarah/yq/releases/download/v4.25.2/yq_linux_amd64 -O /usr/bin/yq
chmod +x /usr/bin/yq
通过 env.rc 脚本配置相关环境变量
```
source env.rc ./clouds.yaml test
```
写入配置其他自定义变量
```
# image 名称
export OPENSTACK_IMAGE_NAME=TestVM

# SSH 密钥对名称(需要提前创建)
export OPENSTACK_SSH_KEY_NAME=node

# 控制节点和计算节点的类型
export OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR=8C-4G
export OPENSTACK_NODE_MACHINE_FLAVOR=4C-2G

# 故障域
export OPENSTACK_FAILURE_DOMAIN=default-az

# openstack external 网络 ID
export OPENSTACK_EXTERNAL_NETWORK_ID=efecd40c-3bc4-401a-bbd8-898894364bd0

# DNS 地址(如果有自己的内部 DNS 最好，如果没有，那么连接 openstack 的所有 endpoint 就需要为可访问的 IP 地址)
export OPENSTACK_DNS_NAMESERVERS=114.114.114.114
```
source 上面的其他自定义变量


172.35.0.2  keystone.openstack.svc.cluster.local
172.35.0.2  nova.openstack.svc.cluster.local
172.35.0.2  neutron.openstack.svc.cluster.local
172.35.0.2  cinder.openstack.svc.cluster.local
172.35.0.2  heat.openstack.svc.cluster.local
172.35.0.2  magnum.eks.svc.cluster.local
172.35.0.2  octavia.octavia.svc.cluster.local
172.38.0.2  hub.ecns.io
172.38.0.2  hub.easystack.cn


clusterctl generate cluster test \
  --kubernetes-version v1.22.8 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 > test.yaml

修改自定义配置test.yaml
kubectl apply -f test.yaml
使用之前
node-4   Ready    master,node   14h   v1.20.14-es   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ceph-mgr=enabled,ceph-mon=enabled,ceph-osd-device-0-node-4=enabled,ceph-osd-device-1-node-4=enabled,ceph-osd=enabled,ceph-rbdmirror=enabled,ceph-rgw=enabled,cluster-type=supervisor,hostha=enabled,kubernetes.io/arch=amd64,kubernetes.io/hostname=node-4,kubernetes.io/os=linux,licensed=true,node-role.kubernetes.io/master=true,node-role.kubernetes.io/node=true,openstack-compute-node=enabled,**openstack-control-plane=enabled**,openstack-network-node=enabled,openvswitch=enabled,roller-control-plane=enabled
