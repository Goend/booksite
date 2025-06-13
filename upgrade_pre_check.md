重点:
# 升级前重点检查项与业务相关风险表

| 检查项目       | 资源类型     | 资源名称                                                       | 集群版本要求     | 处理方法说明                                                                                   |
|----------------|--------------|------------------------------------------------------------------|------------------|------------------------------------------------------------------------------------------------|
| 节点配置检查   | 文件         | /etc/kubernetes/kubelet.env                                     | 所有版本         | 检查文件是否被修改，是否符合当前集群期望配置                                                  |
|    | 文件         | /etc/kubernetes/kubelet_config.yaml                             | 所有版本         | 检查 kubelet 配置参数变更，是否与业务运行兼容                                                  |
|    | 文件         | /etc/containerd/config.toml                                     | 所有版本         | 检查容器运行时配置是否变更，确保 runtime 行为不受影响                                          |
| 资源版本过期检查 | 集群资源     | 所有 Deployment / StatefulSet / CRD 等                          | 下一版本将升级     | 检查当前部署资源定义中使用的 API Version 是否在下个版本中废弃，仅输出警告                       |
|  | Helm Release | 所有已安装 Helm Chart 及其 manifests                            | 下一版本将升级     | 检查模板资源中的 Kubernetes API Version 使用情况，提示是否兼容下一版本                          |
| 兼容性检查     | 集群资源注解 | service.alpha.kubernetes.io/tolerate-unready-endpoints          | >= v1.25         | 替换为 `.spec.publishNotReadyAddresses = true`，否则原注解将失效                                |
|      | 集群资源注解 | service.kubernetes.io/topology-aware-hints    


# 升级前检查项（共 75 条）

1. **节点限制检查**  
   - 检查节点是否可用  node ready 对比华为 https://support.huaweicloud.com/cce_faq/cce_faq_00120.html 实际节点不可用对应Kubernetes 节点没有发送的心跳
   - 操作系统是否支持升级(ecf ecnf操作系统为我们定义 这个应该不需要检查)  skip  ☒
   - 是否含有非预期的节点池标签(没有对节点标签进行强管理) skip ☒
   - K8s 节点名称是否与云服务器一致  待定

2. **升级管控检查**  
   - 检查集群是否处于升级管控状态 华为检查逻辑是1.生产集群被限制升级 需要手动打开 2.进行其他运维任务 不允许升级

3. **插件检查**  
   - 插件状态是否正常 华为检查主要插件功能是否正常 主要包括 coredns,nginx ingress等 待定
   - 是否支持目标版本 由于我们的插件版本本身和eos版本一起升级 检查可能反而引入错误 skip ☒

4. **Helm 模板检查**  
   - 检查 HelmRelease 中是否有目标版本不支持的废弃 API 重点 影响客户体验

5. **控制节点 SSH 连通性检查**  
   - 验证 CCE 能否通过 SSH 连接控制节点 不需要 默认ansible控制所有节点需要以此为前提 skip ☒

6. **节点池检查**  
   - 检查节点池状态是否正常 华为架构中节点池可以伸缩 当处于伸缩中 这里不允许成功 同理 当处于eos扩/缩容时 不允许升级
   - 查升级后节点池操作系统或容器运行时是否支持 由于ecf/ecnf操作系统唯一 因此不考虑此项检查 skip ☒

7. **安全组检查**  
   - 检查 Node 节点安全组中(只针对VPC网络模型) ICMP:全部 且源为控制节点安全组的规则是否被删除 由于我们ecf/ecnf不存在针对裸金属的安全组 因此这段不被需要 skip ☒

8. **残留待迁移节点检查**  
   - 检查节点是否需要迁移 在华为的场景中 该问题由于节点拉包组件异常或节点由比较老的版本升级而来，导致节点上缺少关键的系统组件导致。我们对于这个的需求比较低 skip ☒

9. **K8s 废弃资源检查**  
   - 检查集群是否存在对应版本已经废弃的资源 华为给出的场景主要包括svc的tolerate-unready-endpoints(>=1.25)，service.kubernetes.io/topology-aware-hints(>=1.27) 注解 需要

10. **兼容性风险检查**  
    - 阅读版本兼容性差异并确认不受影响（补丁升级例外）需要

| Source Version | Target Version | Compatibility Differences | Recommended Actions |
|----------------|----------------|---------------------------|---------------------|
| v1.23/v1.25 | v1.27 | Docker container runtime is no longer recommended (use Containerd instead) | Already included in pre-upgrade checks  |
| v1.23 | v1.25 | PodSecurityPolicy removed (replaced by Pod Security Admission) | - Migrate PSP capabilities to PSA <br>- Or delete PSPs before upgrade  |
| v1.21/v1.19 | v1.23 | Nginx Ingress Controller v1.0.0+ requires explicit `kubernetes.io/ingress.class: nginx` annotation | Check Ingress configurations before upgrade  |


11. **节点上 CCE Agent 版本检查**  
    - 是否为最新版本 不需要检查 无相关类似守护进程 skip ☒

12. **节点 CPU 使用率检查**  
    - CPU 使用率是否超过 90% 如果节点当前资源占用很高 不允许升级

13. **CRD 检查**  
    - 检查关键 CRD `packageversions.version.cce.io` 和 `network-attachment-definitions.k8s.cni.cncf.io` 是否被删除  其中前面的crd为cce自定义 但network-attachment-definitions.k8s.cni.cncf.io 在我们的实现中为安全容器中定义  为特定云产品使用并且定义 skip ☒

14. **节点磁盘检查**  
    - 检查关键数据盘使用量及 `/tmp` 目录是否有 ≥500MB 可用空间 不确定escl中是否已经做过类似判断 待定
    - containerd容器运行时磁盘分区（可用空间需满足1G） df -h /var/lib/containerd
    - kubelet磁盘分区（可用空间需满足1G）  df -h /mnt/paas/kubernetes/kubelet
    - 系统盘（可用空间需满足2G）  df -h /


15. **节点 DNS 检查**
    - 节点升级过程中，需要从OBS拉取升级组件包 在我们的场景中 需要检查chart和registry,fs，roller,服务地址是否可以被解析

16. **节点 Kubelet 检查**  
    - Kubelet 服务是否正常运行
    - 检查当前kubelet依赖的pause容器镜像版本是否为cce-pause:3.1 避免容器批量重启

17. **节点内存检查**  
    - 内存使用是否超过 90% 超过不允许升级 避免业务高峰进行升级

18. **节点时钟同步检查**  
    - `ntpd` 或 `chronyd` 是否运行正常 检查服务状态

19. **节点 OS 内核检查**  
    - 内核版本是否被 CCE 支持  内核版本在我们的场景中不允许 不确定是否需要此项检查  待定

20. **节点 CPU 数量检查**  
    - 华为场景检查控制节点 CPU 核心数需大于 2,我们场景中 cpu数目是明确的 skip ☒

21. **节点 Python 命令检查**  
    - Python 命令是否可用 由于我们的实现方式极大概率使用ansible 属于前置 skip ☒

22. **ASM 网格版本检查**  
    - 是否使用 ASM 服务网格  
    - ASM 版本是否支持目标版本  云产品版本check skip ☒

23. **节点 Ready 检查**  
    - 所有节点是否处于 Ready 状态  不需要 这里华为检查的原因是集群本身的状态和kubernetes接口之前还有一层同步 因此再次检查确认 skip ☒

24. **节点 journald 检查**  
    - journald 状态是否正常 systemctl is-active systemd-journald

25. **节点 ContainerdSock 干扰检查**  
    - 检查是否存在干扰的 `containerd.sock` 文件（影响 Euler OS） 排除因为社区版本导致的sock干扰 skip ☒

26. **内部错误检查**  
    - 检查流程中是否出现内部错误 出现内部错误在此项抛出

27. **节点挂载点检查**  
    - 是否存在不可访问挂载点 因为过期的网络存储挂载被访问 访问此目录的进程会处于D  待定
```
for mount_path in `cat /proc/self/mountinfo | awk '{print $5}' | grep -v netns`
do  
    timeout 10 sh -c "cd $mount_path"
    if [ $? == 124 ];then
        echo "$mount_path hang mount"
    fi
done
```

28. **K8s 节点污点检查**  
    - 检查集群升级所需污点是否存在 华为有个升级跳过污点 当环境有此污点 需要重置节点 才能升级成功 我们不存在此场景 skip ☒

29. **Everest 插件版本限制**  
    - 检查当前版本csi插件是否存在兼容性限制  Everest华为云csi插件 由于我们版本升级通常升级过程中 csi插件会在版本升级之后一起升级 因此不存在此场景 skip ☒

30. **cce-hpa-controller 插件限制**  
    - 检查插件目标版本兼容性 无 skip ☒

31. **增强型 CPU 管理策略检查**  
    - 确认两版本是否支持该策略  华为自己开发的策略 基于cpu利用率的优化(容器>85% 会调度到其他使用少的cpu核心) 不存在此场景 skip ☒

32. **用户节点组件健康检查**  
    - 容器运行时、网络组件等是否健康  在我们的场景下 主要是containerd,kube proxy,cni(kube ovn,kube flannel)

33. **控制节点组件健康检查**  
    - Kubernetes、运行时、网络组件等是否正常 在我们的场景下 主要是containerd,kube proxy,cni(kube flannel)

34. **K8s 组件内存资源限制检查**  
    - etcd、controller-manager 等是否超出资源限制 我们不限制etcd,k8s组件对内存的使用 因此此场景不适用 skip ☒

35. **K8s 废弃 API 审计检查**  
    - 扫描过去一天审计日志中是否有废弃 API 调用 检查kubernetes审计日志中是否存在废弃的api被应用 我们没有开启审计日志 无法审计 skip ☒

36. **节点 NetworkManager 检查**  
    - NetworkManager 是否正常运行 systemctl is-active NetworkManager  EOS平面上使用更多的是network escl中应该会检查网络 不确定是否需要此场景 待定

37. **节点 ID 文件检查**  
    - ID 文件内容格式是否正确  无此场景 不需要

38. **节点配置一致性检查**  
    - 升级 v1.19+ 时，检查组件配置是否被后端修改 节点配置一致性检查 主要包括 kubelet启动配置  静态pod配置 容器运行时配置 列出配置并给出修改warn 在华为的场景下 部分参数允许被自定义 但由于升级我们要求强一致 会导致所有非预期更改丢失 因此我们需要给用户提示修改并升级风险

39. **节点配置文件检查**  
    - 检查关键组件配置文件是否存在 主要是kubelet配置文件 静态pod文件

40. **CoreDNS 配置一致性检查**  
    - 检查 Corefile 是否与 Helm Release 记录一致 在我们的场景下 dns在升级的时候会自动覆盖 可能需要提示  待定

41. **节点 Sudo 检查**  
    - sudo 命令及相关文件是否正常 在我们的场景下 不需要 skip ☒

42. **节点关键命令检查**  
    - 升级依赖关键命令是否可正常执行 华为给出两种场景 rpm -qa，systemctl status kubelet   在我们的场景中 待定

43. **节点 sock 文件挂载检查**  
    - 检查是否有 pod 直接挂载 Docker/Containerd 的 sock 文件,给出提示 转变为挂载上层目录

44. **HTTPS 类型 ELB 证书一致性检查**  
    - 检查 ELB 证书是否被修改 不存在此场景 skip ☒

45. **节点挂载检查**  
    - 默认挂载目录或软链接是否被修改或手动挂载 由于华为存在多分区 因此需要检查/etc/fstab文件 在我们的场景下 不需要 skip ☒

46. **节点 paas 用户登录权限检查**  
    - paas 用户是否有登录权限  无此场景 skip ☒

47. **ELB IPv4 私网地址检查**  
    - 检查 ELB 实例是否包含 IPv4 私网 IP  不确定是否需要检查此场景 待定

48. **历史升级记录检查**  
    - 原始版本是否支持目标版本升级 在我们的场景中不存在此场景 skip ☒

49. **管理平面网段一致性检查**  
    - 管理平面网段是否与主干配置一致 管理网段本身应该无法自定义更改 待定

50. **CCE AI（NVIDIA GPU）插件检查**  
    - 由于AI云产品本身的驱动版本维护由用户和对应设备和云产品版本决定 skip ☒

51. **节点系统参数检查**  
    - 系统参数是否被修改 检查bond0网络的mtu值非默认值1500，将出现该检查异常 非容器相关范畴 并且escl网络检查不确定是否有类似检查 待定

52. **残留 packageversion 检查**  
    - 是否存在残留的 packageversion 华为自定义资源 我们是否需要类似的检查 待定

53. **节点命令行检查**  
    - 是否存在升级所需命令 检查依据执行过程中的错误返回 命令不存在的错误 

54. **节点交换区检查**  
    - 是否开启交换区 不需要 在captain中会检查 可以尽早检查 给出错误 待定

55. **NGINX Ingress 插件兼容性检查**  
    - 升级路径是否存在兼容问题 1.检查集群中是否存在未指定Ingress类型（annotations中未添加kubernetes.io/ingress.class: nginx）的Nginx Ingress路由 2.检查Nginx Ingress Controller后端指定的DefaultBackend Service是否存在

56. **云原生监控插件升级检查**  
    - 监控插件升级至 3.9.0 后是否开启 Grafana 开关  云产品 无此场景 skip ☒

57. **Containerd Pod 重启风险检查**  
    - 升级 containerd 时是否可能重启业务容器 没有明确说明的检查方法  待定

58. **CCE AI 插件参数检查**  
    - GPU 插件配置是否被侵入式修改 云产品由于在我们的场景中不属于eos升级部分 无此场景skip ☒

59. **GPU/NPU Pod 重建风险检查**  
    - kubelet 重启时 GPU/NPU 容器是否可能重建 没有明确说明的检查方法 待定

60. **ELB 访问控制配置检查**  
    - 检查当前集群Service是否通过annotation配置了ELB监听器的访问控制 若有配置访问控制则检查相关配置项是否正确 无此场景skip ☒

61. **控制节点规格检查**  
    - 升级前后控制节点规格是否一致 在我们的场景中 规格会发生变化么？ 待定

62. **控制节点子网配额检查**  
    - 子网剩余 IP 是否支持滚动升级 无此场景 skip ☒

63. **节点运行时检查**  
    - 升级至 v1.27+ 后不建议继续使用 Docker 无此场景 skip ☒

64. **节点池运行时检查**  
    - 同上，对节点池运行时的检测  无此场景 skip ☒

65. **节点镜像数量检查**  
    - 镜像数量 >1000 时可能导致启动过慢 待定

66. **OpenKruise 插件兼容性检查**  
    - 检查升级时 OpenKruise 插件兼容情况  无此场景 skip ☒

67. **Secret 落盘加密兼容性检查**  
    - 目标版本是否支持密文落盘特性  无此场景 skip ☒

68. **Ubuntu 内核与 GPU 驱动兼容性提醒**  
    - 若 Ubuntu 内核为 `5.15.0-113-generic`，需 GPU 驱动 ≥ 535.161.08  云产品问题 无此场景 skip ☒

69. **排水任务检查**  
    - 检查是否存在未完成的 drain 任务 无法检查 skip ☒

70. **镜像层数量检查**  
    - 镜像层 >5000 层时可能导致输出延迟 

71. **是否满足滚动升级条件**  
    - 检查集群滚动升级条件是否满足 适用于虚拟机场景 无此场景 skip ☒

72. **证书文件数量检查**  
    - 证书文件 >1000 时可能导致升级缓慢 Pod 被驱逐 无此场景 skip ☒

73. **NetworkPolicy 开关检查**  
    - 检查 NetworkPolicy 配置，如被修改升级时将被重置 目前对于networkpolicy没有明确支持(kube ovn) 无此场景 skip ☒

74. **集群与节点池配置管理检查**  
    - 检查 `nic-max-above-warm-target` 是否超过最大允许值 无此场景 skip ☒

75. **控制节点时区检查**  
    - 检查控制节点时区是否与集群时区一致，升级后将统一时区 时间同步检查在escl中应该做了 需要确认 待定
