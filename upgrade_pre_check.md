# 升级前检查项（共 75 条）

1. **节点限制检查**  
   - 检查节点是否可用  
   - 操作系统是否支持升级  
   - 是否含有非预期的节点池标签  
   - K8s 节点名称是否与云服务器一致 :contentReference[oaicite:1]{index=1}

2. **升级管控检查**  
   - 检查集群是否处于升级管控状态 :contentReference[oaicite:2]{index=2}

3. **插件检查**  
   - 插件状态是否正常  
   - 是否支持目标版本 :contentReference[oaicite:3]{index=3}

4. **Helm 模板检查**  
   - 检查 HelmRelease 中是否有目标版本不支持的废弃 API :contentReference[oaicite:4]{index=4}

5. **控制节点 SSH 连通性检查**  
   - 验证 CCE 能否通过 SSH 连接控制节点 :contentReference[oaicite:5]{index=5}

6. **节点池检查**  
   - 检查节点池状态是否正常 :contentReference[oaicite:6]{index=6}

7. **安全组检查**  
   - 检查 Node 节点安全组中 ICMP:全部 且源为控制节点安全组的规则是否被删除 :contentReference[oaicite:7]{index=7}

8. **残留待迁移节点检查**  
   - 检查节点是否需要迁移 :contentReference[oaicite:8]{index=8}

9. **K8s 废弃资源检查**  
   - 检查资源是否存在对应版本已废弃的情况 :contentReference[oaicite:9]{index=9}

10. **兼容性风险检查**  
    - 阅读版本兼容性差异并确认不受影响（补丁升级例外） :contentReference[oaicite:10]{index=10}

11. **节点上 CCE Agent 版本检查**  
    - 是否为最新版本 :contentReference[oaicite:11]{index=11}

12. **节点 CPU 使用率检查**  
    - CPU 使用率是否超过 90% :contentReference[oaicite:12]{index=12}

13. **CRD 检查**  
    - 检查关键 CRD `packageversions.version.cce.io` 和 `network-attachment-definitions.k8s.cni.cncf.io` 是否被删除 :contentReference[oaicite:13]{index=13}

14. **节点磁盘检查**  
    - 检查关键数据盘使用量及 `/tmp` 目录是否有 ≥500MB 可用空间 :contentReference[oaicite:14]{index=14}

15. **节点 DNS 检查**  
    - DNS 是否能解析 OBS 地址  
    - 是否能访问 OBS 下载包 :contentReference[oaicite:15]{index=15}

16. **节点关键目录文件权限检查**  
    - `/var/paas` 文件属主和属组是否为 `paas` :contentReference[oaicite:16]{index=16}

17. **节点 Kubelet 检查**  
    - Kubelet 服务是否正常运行 :contentReference[oaicite:17]{index=17}

18. **节点内存检查**  
    - 内存使用是否超过 90% :contentReference[oaicite:18]{index=18}

19. **节点时钟同步检查**  
    - `ntpd` 或 `chronyd` 是否运行正常 :contentReference[oaicite:19]{index=19}

20. **节点 OS 内核检查**  
    - 内核版本是否被 CCE 支持 :contentReference[oaicite:20]{index=20}

21. **节点 CPU 数量检查**  
    - 控制节点 CPU 核心数需大于 2 :contentReference[oaicite:21]{index=21}

22. **节点 Python 命令检查**  
    - Python 命令是否可用 :contentReference[oaicite:22]{index=22}

23. **ASM 网格版本检查**  
    - 是否使用 ASM 服务网格  
    - ASM 版本是否支持目标版本 :contentReference[oaicite:23]{index=23}

24. **节点 Ready 检查**  
    - 所有节点是否处于 Ready 状态 :contentReference[oaicite:24]{index=24}

25. **节点 journald 检查**  
    - journald 状态是否正常 :contentReference[oaicite:25]{index=25}

26. **节点 ContainerdSock 干扰检查**  
    - 检查是否存在干扰的 `containerd.sock` 文件（影响 Euler OS） :contentReference[oaicite:26]{index=26}

27. **内部错误检查**  
    - 检查流程中是否出现内部错误 :contentReference[oaicite:27]{index=27}

28. **节点挂载点检查**  
    - 是否存在不可访问挂载点 :contentReference[oaicite:28]{index=28}

29. **K8s 节点污点检查**  
    - 检查集群升级所需污点是否存在 :contentReference[oaicite:29]{index=29}

30. **Everest 插件版本限制**  
    - 检查当前版本是否存在兼容性限制 :contentReference[oaicite:30]{index=30}

31. **cce-hpa-controller 插件限制**  
    - 检查插件目标版本兼容性 :contentReference[oaicite:31]{index=31}

32. **增强型 CPU 管理策略检查**  
    - 确认两版本是否支持该策略 :contentReference[oaicite:32]{index=32}

33. **用户节点组件健康检查**  
    - 容器运行时、网络组件等是否健康 :contentReference[oaicite:33]{index=33}

34. **控制节点组件健康检查**  
    - Kubernetes、运行时、网络组件等是否正常 :contentReference[oaicite:34]{index=34}

35. **K8s 组件内存资源限制检查**  
    - etcd、controller-manager 等是否超出资源限制 :contentReference[oaicite:35]{index=35}

36. **K8s 废弃 API 审计检查**  
    - 扫描过去一天审计日志中是否有废弃 API 调用 :contentReference[oaicite:36]{index=36}

37. **节点 NetworkManager 检查**  
    - NetworkManager 是否正常运行 :contentReference[oaicite:37]{index=37}

38. **节点 ID 文件检查**  
    - ID 文件内容格式是否正确 :contentReference[oaicite:38]{index=38}

39. **节点配置一致性检查**  
    - 升级 v1.19+ 时，检查组件配置是否被后端修改 :contentReference[oaicite:39]{index=39}

40. **节点配置文件检查**  
    - 检查关键组件配置文件是否存在 :contentReference[oaicite:40]{index=40}

41. **CoreDNS 配置一致性检查**  
    - 检查 Corefile 是否与 Helm Release 记录一致 :contentReference[oaicite:41]{index=41}

42. **节点 Sudo 检查**  
    - sudo 命令及相关文件是否正常 :contentReference[oaicite:42]{index=42}

43. **节点关键命令检查**  
    - 升级依赖关键命令是否可正常执行 :contentReference[oaicite:43]{index=43}

44. **节点 sock 文件挂载检查**  
    - 检查是否有 pod 直接挂载 Docker/Containerd 的 sock 文件 :contentReference[oaicite:44]{index=44}

45. **HTTPS 类型 ELB 证书一致性检查**  
    - 检查 ELB 证书是否被修改 :contentReference[oaicite:45]{index=45}

46. **节点挂载检查**  
    - 默认挂载目录或软链接是否被修改或手动挂载 :contentReference[oaicite:46]{index=46}

47. **节点 paas 用户登录权限检查**  
    - paas 用户是否有登录权限 :contentReference[oaicite:47]{index=47}

48. **ELB IPv4 私网地址检查**  
    - 检查 ELB 实例是否包含 IPv4 私网 IP :contentReference[oaicite:48]{index=48}

49. **历史升级记录检查**  
    - 原始版本是否支持目标版本升级 :contentReference[oaicite:49]{index=49}

50. **管理平面网段一致性检查**  
    - 管理平面网段是否与主干配置一致 :contentReference[oaicite:50]{index=50}

51. **CCE AI（NVIDIA GPU）插件检查**  
    - 升级是否影响 GPU 驱动安装 :contentReference[oaicite:51]{index=51}

52. **节点系统参数检查**  
    - 系统参数是否被修改 :contentReference[oaicite:52]{index=52}

53. **残留 packageversion 检查**  
    - 是否存在残留的 packageversion :contentReference[oaicite:53]{index=53}

54. **节点命令行检查**  
    - 是否存在升级所需命令 :contentReference[oaicite:54]{index=54}

55. **节点交换区检查**  
    - 是否开启交换区 :contentReference[oaicite:55]{index=55}

56. **NGINX Ingress 插件兼容性检查**  
    - 升级路径是否存在兼容问题 :contentReference[oaicite:56]{index=56}

57. **云原生监控插件升级检查**  
    - 监控插件升级至 3.9.0 后是否开启 Grafana 开关 :contentReference[oaicite:57]{index=57}

58. **Containerd Pod 重启风险检查**  
    - 升级 containerd 时是否可能重启业务容器 :contentReference[oaicite:58]{index=58}

59. **CCE AI 插件参数检查**  
    - GPU 插件配置是否被侵入式修改 :contentReference[oaicite:59]{index=59}

60. **GPU/NPU Pod 重建风险检查**  
    - kubelet 重启时 GPU/NPU 容器是否可能重建 :contentReference[oaicite:60]{index=60}

61. **ELB 访问控制配置检查**  
    - 若有访问控制，检查其配置是否正确 :contentReference[oaicite:61]{index=61}

62. **控制节点规格检查**  
    - 升级前后控制节点规格是否一致 :contentReference[oaicite:62]{index=62}

63. **控制节点子网配额检查**  
    - 子网剩余 IP 是否支持滚动升级 :contentReference[oaicite:63]{index=63}

64. **节点运行时检查**  
    - 升级至 v1.27+ 后不建议继续使用 Docker :contentReference[oaicite:64]{index=64}

65. **节点池运行时检查**  
    - 同上，对节点池运行时的检测 :contentReference[oaicite:65]{index=65}

66. **节点镜像数量检查**  
    - 镜像数量 >1000 时可能导致启动过慢 :contentReference[oaicite:66]{index=66}

67. **OpenKruise 插件兼容性检查**  
    - 检查升级时 OpenKruise 插件兼容情况 :contentReference[oaicite:67]{index=67}

68. **Secret 落盘加密兼容性检查**  
    - 目标版本是否支持密文落盘特性 :contentReference[oaicite:68]{index=68}

69. **Ubuntu 内核与 GPU 驱动兼容性提醒**  
    - 若 Ubuntu 内核为 `5.15.0-113-generic`，需 GPU 驱动 ≥ 535.161.08 :contentReference[oaicite:69]{index=69}

70. **排水任务检查**  
    - 检查是否存在未完成的 drain 任务 :contentReference[oaicite:70]{index=70}

71. **镜像层数量检查**  
    - 镜像层 >5000 层时可能导致输出延迟 :contentReference[oaicite:71]{index=71}

72. **是否满足滚动升级条件**  
    - 检查集群滚动升级条件是否满足 :contentReference[oaicite:72]{index=72}

73. **证书文件数量检查**  
    - 证书文件 >1000 时可能导致升级缓慢 Pod 被驱逐 :contentReference[oaicite:73]{index=73}

74. **NetworkPolicy 开关检查**  
    - 检查 NetworkPolicy 配置，如被修改升级时将被重置 :contentReference[oaicite:74]{index=74}

75. **集群与节点池配置管理检查**  
    - 检查 `nic-max-above-warm-target` 是否超过最大允许值 :contentReference[oaicite:75]{index=75}

76. **控制节点时区检查**  
    - 检查控制节点时区是否与集群时区一致，升级后将统一时区 :contentReference[oaicite:76]{index=76}
