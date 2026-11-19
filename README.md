# 在 EKS 上用 Graviton 节点部署 ClickHouse

本文档根据在真实 EKS 集群上的两轮安装测试整理。第一轮验证了基本安装,第二轮把第一轮没测的部分(持久化存储、高可用、污点隔离、混合架构、官方 operator)补测了一遍。文档里的命令和清单,除第 9 节明确列出的以外,都实际运行过。

## 1. 结论

ClickHouse 可以在 EKS 的 Graviton 节点上部署和运行,以下几项都实测通过:

- Altinity operator、ClickHouse Keeper、ClickHouse 全部运行在 arm64 节点上。
- 数据放在 EBS gp3 卷上,删除 pod 后数据不丢。
- 3 副本 Keeper 加 2 分片 × 2 副本 ClickHouse,pod 分散在不同节点,挂掉一个 Keeper 或一个 ClickHouse 副本时读写不受影响。
- 同一分片里 arm64 副本和 x86 副本可以互相复制,数据一致,因此可以逐个替换副本迁移到 Graviton。
- ClickHouse 官方 operator 也能在 arm64 上工作。

需要留意的不是 ClickHouse 本身,而是集群里的周边组件和几个配置细节,见第 8 节。

## 2. 测试环境

| 项 | 值 |
|---|---|
| 区域 / 集群 | us-east-1,EKS `eks-test`,Kubernetes 1.32 |
| Graviton 节点 | 1 × c8g.8xlarge(32 vCPU / 64 GB);4 × m8g.large(带污点,用于高可用验证);都是 AL2023 arm64,kubelet v1.32.13 |
| x86 节点 | 2 × t3.medium(集群原有,用于混合架构验证) |
| Operator | Altinity clickhouse-operator 0.27.4;ClickHouse 官方 operator v0.0.8 |
| ClickHouse / Keeper | `clickhouse/clickhouse-server:25.8`、`clickhouse/clickhouse-keeper:25.8`(实际版本 25.8.33.6) |
| 存储 | EBS CSI 插件 v1.68.0-eksbuild.1,gp3 |
| 客户端工具 | kubectl v1.32.9,helm v3.19.0 |

## 3. 前置条件

### 3.1 CPU 要求

ClickHouse 官方 arm64 镜像要求 ARMv8.2-A 加 RCpc 扩展,官方文档写明支持 Graviton 2 及以上,Graviton 1(A1 实例)不能用。测试中从 `system.build_options` 读到的编译参数是 `-march=armv8.2-a+simd+crypto+dotprod+ssbs+rcpc+bf16`,与文档一致。

### 3.2 镜像架构

以下镜像都在 arm64 节点上实际运行过:

| 镜像 | 版本 |
|---|---|
| `altinity/clickhouse-operator`、`altinity/metrics-exporter` | 0.27.4 |
| `clickhouse/clickhouse-server`、`clickhouse/clickhouse-keeper` | 25.8 |
| `ghcr.io/clickhouse/clickhouse-operator` | v0.0.8 |
| cert-manager(官方 operator 的依赖) | v1.21.3 |
| EBS CSI 插件 | v1.68.0-eksbuild.1 |

### 3.3 集群权限

操作者需要集群管理员权限(要装 CRD)。EKS 用 access entry 授权:

```bash
aws eks create-access-entry --region <region> --cluster-name <cluster> \
  --principal-arn <你的 IAM 角色 ARN>

aws eks associate-access-policy --region <region> --cluster-name <cluster> \
  --principal-arn <你的 IAM 角色 ARN> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

### 3.4 检查现有插件是否支持 arm64

**加 Graviton 节点之前先做这一步。** 测试集群原有的 EFS CSI 插件(v2.1.2)在 arm64 节点上启动失败,导致插件状态变成 DEGRADED(详见第 8 节)。新节点加入后检查:

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=<新的 arm64 节点名>
```

所有 DaemonSet 的 pod 都应该是 Running。

## 4. Operator 选型

| | Altinity operator | ClickHouse 官方 operator |
|---|---|---|
| 成熟度 | 发布多年,当前 0.27 / 0.28 | API 版本 `v1alpha1`,chart 版本 0.0.8 |
| CRD | `ClickHouseInstallation`、`ClickHouseKeeperInstallation` | `ClickHouseCluster`、`KeeperCluster` |
| 依赖 | 无 | cert-manager |
| 本次测试 | 基本安装、持久化、高可用、混合架构都测过 | 测了基本安装、gp3 持久卷、2 副本复制 |

本文档以 Altinity operator 为主。官方 operator 在 arm64 上能正常工作,但 API 还在 alpha 阶段,后续 CRD 可能有不兼容变更,建议只用于 PoC。官方 operator 的安装见第 7.5 节。

## 5. 基本部署步骤

### 5.1 创建 Graviton 节点组

```bash
aws eks create-nodegroup --region us-east-1 \
  --cluster-name eks-test \
  --nodegroup-name ch-graviton-test \
  --instance-types c8g.8xlarge \
  --ami-type AL2023_ARM_64_STANDARD \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=1,maxSize=1,desiredSize=1 \
  --disk-size 100 \
  --subnets <subnet-1> <subnet-2> \
  --node-role <节点 IAM 角色 ARN> \
  --labels workload=clickhouse-test
```

要点:

- `--ami-type` 必须选 ARM 的 AMI 类型(`AL2023_ARM_64_STANDARD`),不能沿用 x86 节点组的设置。
- 节点角色可以沿用现有节点组的角色。
- 测试中从发起创建到节点组变为 ACTIVE 不到 3 分钟。

> 测试时节点组是通过 EKS API 用同样的参数创建的,上面的 CLI 命令是等价写法,命令本身没有单独执行过。

确认节点:

```bash
kubectl get nodes -L kubernetes.io/arch,node.kubernetes.io/instance-type,workload
```

### 5.2 安装 operator

```bash
helm repo add altinity https://docs.altinity.com/clickhouse-operator/
helm repo update

helm install clickhouse-operator altinity/altinity-clickhouse-operator --version 0.27.4 \
  -n clickhouse-test --create-namespace \
  --set-string nodeSelector."kubernetes\.io/arch"=arm64 \
  --set-string nodeSelector.workload=clickhouse-test \
  --set-string crdHook.nodeSelector."kubernetes\.io/arch"=arm64 \
  --set-string crdHook.nodeSelector.workload=clickhouse-test \
  --wait --timeout 5m
```

说明:

- 两组 `nodeSelector` 分别作用于 operator 本身和安装 CRD 的 Job,这样 operator 也跑在 Graviton 节点上。
- **operator 默认只监听自己所在的命名空间。** 如果 ClickHouse 要部署到别的命名空间,必须设置 `watchNamespaces`,否则 apply 之后没有任何反应,也没有报错:

```bash
helm upgrade clickhouse-operator altinity/altinity-clickhouse-operator --version 0.27.4 \
  -n clickhouse-test --reuse-values \
  --set 'watchNamespaces={clickhouse-test,clickhouse}' --wait
```

- 截至测试时,Docker Hub 上已有 0.28.0 镜像,但 Helm 仓库里最新的 chart 是 0.27.4。

验证:

```bash
kubectl -n clickhouse-test get pods          # operator pod 应为 2/2 Running
kubectl get crd | grep -i clickhouse         # 应有 4 个 altinity.com 的 CRD
```

### 5.3 部署 Keeper,再部署 ClickHouse

清单分成两个文件:`manifests/01-keeper.yaml`(Keeper,1 副本)和 `manifests/02-clickhouse.yaml`(ClickHouse 集群,1 分片 2 副本)。这一组用的是容器本地盘,只用于快速验证安装;带持久卷的部署见第 7 节。

**先部署 Keeper 并等它就绪,再部署 ClickHouse:**

```bash
# 1) Keeper
kubectl apply -f manifests/01-keeper.yaml
kubectl -n clickhouse-test get chk graviton-keeper -w      # 等到 STATUS = Completed

# 2) ClickHouse
kubectl apply -f manifests/02-clickhouse.yaml
kubectl -n clickhouse-test get chi graviton -w             # 等到 STATUS = Completed
```

顺序的影响实测过两次:

- 两者一起 apply 时,ClickHouse 比 Keeper 先起来,日志里报几条 `Host not found: keeper-graviton-keeper`,Keeper 就绪后自动恢复。
- 先 Keeper 后 ClickHouse 时(第 7 节的高可用集群),4 个 ClickHouse pod 的启动日志里 Keeper 相关报错为 0。

清单里和 Graviton 相关的关键配置:

```yaml
nodeSelector:
  kubernetes.io/arch: arm64
  workload: clickhouse-test
```

ClickHouse 连 Keeper 用的 Service 名是 `keeper-<Keeper 资源名>`,端口 2181:

```yaml
zookeeper:
  nodes:
    - host: keeper-graviton-keeper
      port: 2181
```

## 6. 验证

### 6.1 确认跑在 arm64 上

```bash
kubectl -n clickhouse-test exec chi-graviton-test-0-0-0 -c clickhouse -- \
  clickhouse-client -q "SELECT version(), (SELECT value FROM system.build_options WHERE name='SYSTEM_PROCESSOR')"
```

测试输出:`25.8.33.6  aarch64`。

### 6.2 确认集群拓扑

```bash
kubectl -n clickhouse-test exec chi-graviton-test-0-0-0 -c clickhouse -- \
  clickhouse-client -q "SELECT cluster, shard_num, replica_num, host_name FROM system.clusters WHERE cluster='test'"
```

应该看到全部副本。**如果行数不够,等一两分钟再查**:operator 状态显示 Completed 之后,更新的 ConfigMap 还要等 kubelet 同步到 pod 里才生效。这个延迟两轮测试都出现了,第一轮约 2 分钟,第二轮不到 1 分钟。在拓扑同步完成之前执行 `ON CLUSTER` 的 DDL,只会落到部分副本上。

### 6.3 确认 Keeper 和复制

```sql
-- 在副本 0 上执行
CREATE TABLE default.t ON CLUSTER test (id UInt64, k UInt32, v Float64)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/t', '{replica}')
ORDER BY id;

INSERT INTO default.t SELECT number, number % 1000, rand() / 4294967295 FROM numbers(100000000);

-- 在副本 1 上执行
SYSTEM SYNC REPLICA default.t;
SELECT count() FROM default.t;      -- 应为 100000000

SELECT replica_name, is_readonly, queue_size, active_replicas, total_replicas
FROM system.replicas WHERE table = 't';
```

第一轮测试结果(单台 c8g.8xlarge,本地盘):

| 项 | 结果 |
|---|---|
| 副本 0 写 1 亿行,副本 1 读 | 1 亿行 |
| 副本 1 再写 100 万行,副本 0 读 | 1.01 亿行 |
| 复制状态 | `is_readonly=0`,`queue_size=0`,`active_replicas=2/2` |
| 1 亿行写入本地 MergeTree | 2.81 秒 |
| 1.01 亿行 `GROUP BY` + `avg` + `quantile(0.99)` | 三次约 0.09 秒 |

写入和聚合的耗时是单次粗测(合成数据、本地盘、单节点),只说明功能正常,不能当基准成绩,也不能用来和 x86 对比。

## 7. 生产相关配置

本节内容第二轮都实测过。对应清单是 `manifests/clickhouse-production-example.yaml`,测试时把规格缩小了(ClickHouse 250m CPU / 3Gi 内存 / 20Gi 卷),拓扑、反亲和、容忍、存储配置与清单一致;清单里 28 vCPU / 48Gi / 500Gi 的数值没有按原尺寸跑过。

### 7.1 持久化存储

**现象**:集群没有 EBS CSI 插件时,PVC 一直 Pending,事件里是 `Waiting for a volume to be created either by the external provisioner 'ebs.csi.aws.com'`。集群默认的 `gp2` StorageClass 虽然写的是旧的 in-tree provisioner(`kubernetes.io/aws-ebs`),实际也要靠 EBS CSI 插件才能建卷;插件装好后,之前 Pending 的 gp2 PVC 自动绑定成功。

**步骤:**

1. 给 EBS CSI 插件建一个 IAM 角色,信任 EKS Pod Identity,挂 `AmazonEBSCSIDriverPolicy`:

```bash
aws iam create-role --role-name AmazonEKSPodIdentityAmazonEBSCSIRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

aws iam attach-role-policy --role-name AmazonEKSPodIdentityAmazonEBSCSIRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

2. 安装插件并关联角色(集群需要已安装 `eks-pod-identity-agent` 插件):

```bash
aws eks create-addon --region us-east-1 --cluster-name eks-test \
  --addon-name aws-ebs-csi-driver --addon-version v1.68.0-eksbuild.1 \
  --pod-identity-associations serviceAccount=ebs-csi-controller-sa,roleArn=arn:aws:iam::<账号>:role/AmazonEKSPodIdentityAmazonEBSCSIRole
```

3. 创建 gp3 StorageClass:

```bash
kubectl apply -f manifests/gp3-storageclass.yaml
```

4. 在 CHI / CHK 里加 `volumeClaimTemplates` 并在 `defaults.templates.dataVolumeClaimTemplate` 里引用,见生产示例清单。

> 第 1、2 步测试时是通过 IAM 和 EKS 的 API 用同样的参数完成的,上面的 CLI 命令是等价写法,命令本身没有单独执行过。

**实测结果:**

- 插件的 controller 和 node pod 在 arm64 节点上全部正常运行。
- 3 个 Keeper 卷、4 个 ClickHouse 卷全部绑定,`/var/lib/clickhouse` 挂在 EBS 卷上。
- 删除一个 ClickHouse 副本 pod,新 pod 挂回原来的卷,该副本的 5,000,171 行数据原样还在。

不要把 ClickHouse 数据放在 EFS 上。IO 特别重的场景可以考虑带本地 NVMe 的 Graviton 机型(如 i8g、im4gn),但本地盘在节点替换时数据会丢,需要靠副本恢复;这类机型本次没有测试。

### 7.2 高可用

生产示例的拓扑:Keeper 3 副本,ClickHouse 2 分片 × 2 副本,用 pod 反亲和(`topologyKey: kubernetes.io/hostname`)让同一套集群的 pod 互不同机。

部署顺序同样是先 Keeper 后 ClickHouse。生产示例是一个文件里的两段,用 `---` 分隔,拆成两个文件分别 apply 即可。

**实测结果:**

| 项 | 结果 |
|---|---|
| Keeper 3 个 pod | 落在 3 台不同节点 |
| ClickHouse 4 个 pod | 落在 4 台不同节点 |
| 部署耗时 | Keeper 约 90 秒,ClickHouse 约 110 秒 |
| 分布式表写入 1000 万行 | 两个分片各约 500 万行,每个分片的两个副本行数一致,任意 pod 查分布式表都是 1000 万行 |
| 删掉 1 个 Keeper pod(完全消失后立即写入) | 写入成功,另一副本同步到数据 |
| 删掉 1 个 ClickHouse 副本 pod | 重建期间通过其他副本查询正常,重建后数据完整 |

**两个限制:**

- **节点数要求**:按主机名做反亲和时,每个 ClickHouse pod 独占一台节点。2 分片 × 2 副本至少要 4 台 arm64 节点,Keeper 至少 3 台(Keeper 可以和 ClickHouse 同机)。
- **不保证跨可用区**:按主机名的反亲和只保证不同机,不保证不同可用区。测试中分片 1 的两个副本都落在了 us-east-1b。要让同一分片的副本分散到不同可用区,需要另外加按可用区的调度约束(如 `topologySpreadConstraints`),并保证每个可用区有足够的节点;这一点本次没有验证。

### 7.3 节点隔离

用污点把 ClickHouse 节点和其他工作负载隔开。节点组加污点:

```bash
aws eks create-nodegroup ... \
  --taints key=dedicated,value=clickhouse,effect=NO_SCHEDULE
```

pod 模板里加容忍:

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: clickhouse
    effect: NoSchedule
```

实测:带容忍的 Keeper 和 ClickHouse pod 正常调度到了带污点的节点上。

注意两点:

- 污点只负责把别的 pod 挡在外面,不负责把 ClickHouse 吸过来。生产示例只按架构选节点(`kubernetes.io/arch: arm64`),测试中有 1 个 ClickHouse pod 落到了另一台没有污点的 arm64 节点上。要把 ClickHouse 限定在专用节点组,`nodeSelector` 里还要加节点组的标签。
- operator 自己没有加容忍,要么让它跑在没有污点的节点上,要么在 Helm 里给它也配上 `tolerations`。

### 7.4 混合架构和迁移

同一分片里 arm64 副本和 x86 副本可以共存并互相复制。清单见 `manifests/mixed-arch-example.yaml`:一个分片两个副本,通过给每个副本指定不同的 pod 模板,让副本 0 跑在 Graviton 上,副本 1 跑在 x86 上。

**实测结果**(表里包含 LowCardinality、Decimal、DateTime64、数组,以及 Gorilla、DoubleDelta、ZSTD、LZ4 编码):

| 项 | 结果 |
|---|---|
| arm64 写 500 万行,x86 读 | 一致 |
| x86 写 200 万行,arm64 读 | 一致 |
| 两边的行数和全列哈希(`sum(cityHash64(*))`) | 完全相同 |
| 在 x86 上执行 `OPTIMIZE ... FINAL` 合并数据片段(part),arm64 同步 | 两边都剩 1 个数据片段,哈希仍然相同 |
| 日志里的校验和不匹配报错 | 0 |

`sum()` 一个浮点列时,两边结果在小数点后第 8 位以后有差异。我判断这是浮点数并行求和顺序不同造成的,不是数据不一致:全列哈希是相同的,而且同一个副本在合并前后两次查询的结果也有同样量级的差异。

**迁移思路**:给现有 x86 分片加一个 arm64 副本,等同步完成后再下线 x86 副本,逐个替换。测试验证的是"混合架构下复制正常且数据一致",完整的迁移流程(在线加副本、切流量、下线旧副本)没有演练。

如果集群里同时有 x86 和 Graviton 节点,ClickHouse 和 Keeper 的 pod 模板必须带 `nodeSelector: kubernetes.io/arch`。镜像是多架构的,不加也能在任意节点跑起来,pod 会被随机调度到某种架构上。

### 7.5 ClickHouse 官方 operator

```bash
# 依赖:cert-manager
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  -n cert-manager --create-namespace --set crds.enabled=true \
  --set-string nodeSelector."kubernetes\.io/arch"=arm64 \
  --set-string webhook.nodeSelector."kubernetes\.io/arch"=arm64 \
  --set-string cainjector.nodeSelector."kubernetes\.io/arch"=arm64 \
  --set-string startupapicheck.nodeSelector."kubernetes\.io/arch"=arm64 \
  --wait --timeout 6m

# operator
helm install clickhouse-operator oci://ghcr.io/clickhouse/clickhouse-operator-helm --version 0.0.8 \
  -n clickhouse-operator-system --create-namespace \
  --set-string manager.nodeSelector."kubernetes\.io/arch"=arm64 --wait --timeout 6m

# 集群
kubectl create namespace clickhouse-official
kubectl apply -f manifests/official-operator-example.yaml
```

实测结果:

- cert-manager 和 operator 都在 arm64 节点上正常运行。
- 1 副本 Keeper 加 1 分片 2 副本 ClickHouse 约 50 秒就绪,数据卷是 gp3。
- 在 Replicated 数据库里建表、写 100 万行,两个副本行数一致。
- Keeper 和 ClickHouse 一起 apply 时,两个 ClickHouse pod 各重启了 2 次才起来,原因是启动时 Keeper 还没就绪(`Cannot use any of provided ZooKeeper nodes`)。
- 清单没有设置 `spec.settings.defaultUserPassword`,apply 时会警告 default 用户没有密码。

官方 operator 和 Altinity operator 的 CRD 不同组(`clickhouse.com` 和 `clickhouse.altinity.com`),可以装在同一个集群里,测试中两者同时运行没有冲突。

## 8. 测试中遇到的问题

| 问题 | 现象 | 原因和处理 |
|---|---|---|
| EFS CSI 插件在 arm64 节点崩溃 | 每台新的 arm64 节点上 `efs-csi-node` 都 CrashLoopBackOff,日志 `exec /bin/aws-efs-csi-driver: exec format error`;EKS 把该插件标记为 DEGRADED | 集群原有插件版本是 v2.1.2,当前 1.32 的默认版本是 v3.6.0。测试中没有处理。影响是 arm64 节点上挂不了 EFS 卷,不影响 ClickHouse。升级插件能否解决没有验证 |
| operator 不处理其他命名空间的资源 | 在别的命名空间 apply CHK / CHI 后状态一直为空,没有 pod,也没有报错 | chart 默认 `watchNamespaces: []`,只监听 operator 自己所在的命名空间。按 5.2 设置后立即开始处理 |
| 没有 EBS CSI 插件时建不出卷 | PVC 一直 Pending | 按 7.1 安装插件 |
| 启动初期 Keeper 连接报错 | Altinity:日志报 `Host not found`;官方 operator:pod 重启 2 次 | ClickHouse 比 Keeper 先起来。先部署 Keeper 可避免 |
| `ON CLUSTER` 建表只落到部分副本 | 其他副本上查表报 `UNKNOWN_TABLE` | 拓扑配置同步有 2 分钟左右的延迟。等 `system.clusters` 显示完整后再执行 DDL;已发生的话用 `CREATE TABLE IF NOT EXISTS ... ON CLUSTER` 补建 |
| 按主机名反亲和不保证跨可用区 | 一个分片的两个副本落在同一可用区 | 见 7.2 |
| 只按架构选节点时 pod 会落到别的 arm64 节点 | 1 个 ClickHouse pod 调度到了专用节点组之外 | `nodeSelector` 里加节点组标签,见 7.3 |
| kubectl 返回 Unauthorized | 操作用的 IAM 角色不在集群 access entries 里 | 按 3.3 添加 access entry |

Altinity operator 的 CHI 状态里每次都会记录几条 `Update Service ... failed with error: just recreate the service in case of service type change`。Service 实际都创建成功,集群功能不受影响,原因没有查。

## 9. 仍然没有验证的部分

- **生产规格**:生产示例里的 28 vCPU / 48Gi / 500Gi 没有按原尺寸运行,只用缩小的规格验证了结构。
- **性能**:没有做基准测试,没有和 x86 做性能对比。
- **跨可用区分散**:没有验证按可用区的调度约束。
- **节点和可用区故障**:只测了删除 pod,没有测节点宕机、可用区不可用、EBS 卷故障。
- **完整迁移流程**:只验证了混合架构下复制正常,没有演练 x86 到 Graviton 的在线迁移。
- **日常运维**:版本升级、扩缩分片和副本、卷扩容、备份恢复、监控告警都没有测。
- **EFS CSI 插件升级**:没有验证升级后能否在 arm64 上运行。
- **资源回收**:删除 CHI / CHK 之后 PVC 和 EBS 卷是否自动删除没有验证。
- **本地 NVMe 机型**(i8g、im4gn 等)没有测试。
- **第 5.1、7.1、7.3 节的 AWS CLI 命令**:对应操作是通过 API 完成的,CLI 写法没有单独执行。
- **第 10 节的清理命令**全部没有执行过。

## 10. 清理

以下命令没有执行过,测试环境目前仍在运行。

```bash
# 1. 官方 operator 的集群和 operator
kubectl delete -n clickhouse-official clickhousecluster official
kubectl delete -n clickhouse-official keepercluster official-keeper
kubectl delete ns clickhouse-official
helm uninstall clickhouse-operator -n clickhouse-operator-system
kubectl delete ns clickhouse-operator-system
kubectl get crd -o name | grep -E '\.clickhouse\.com$' | xargs kubectl delete
helm uninstall cert-manager -n cert-manager
kubectl delete ns cert-manager

# 2. Altinity 的集群(先删 CHI / CHK,再删 operator)
kubectl delete -n clickhouse chi prod
kubectl delete -n clickhouse chk prod-keeper
kubectl delete -n clickhouse-test chi mixed graviton
kubectl delete -n clickhouse-test chk graviton-keeper
kubectl delete -n clickhouse-test pod gp2-probe
kubectl delete -n clickhouse-test pvc gp2-probe

# 3. 确认 PVC 都删干净(否则 EBS 卷会留下继续计费)
kubectl get pvc -A
kubectl delete pvc --all -n clickhouse
kubectl delete pvc --all -n clickhouse-official

# 4. Altinity operator 和命名空间
helm uninstall clickhouse-operator -n clickhouse-test
kubectl delete ns clickhouse clickhouse-test
kubectl get crd -o name | grep -E 'clickhouse(-keeper)?\.altinity\.com' | xargs kubectl delete
kubectl delete storageclass gp3

# 5. 节点组
aws eks delete-nodegroup --region us-east-1 --cluster-name eks-test --nodegroup-name ch-graviton-ha-test
aws eks delete-nodegroup --region us-east-1 --cluster-name eks-test --nodegroup-name ch-graviton-test

# 6. EBS CSI 插件和它的 IAM 角色(如果集群以后不需要 EBS 卷)
aws eks delete-addon --region us-east-1 --cluster-name eks-test --addon-name aws-ebs-csi-driver
aws iam detach-role-policy --role-name AmazonEKSPodIdentityAmazonEBSCSIRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
aws iam delete-role --role-name AmazonEKSPodIdentityAmazonEBSCSIRole

# 7. 收回测试时添加的集群管理员权限
aws eks delete-access-entry --region us-east-1 --cluster-name eks-test \
  --principal-arn <测试时授权的 IAM 角色 ARN>
```

第 3 步要在删节点组和 EBS CSI 插件之前做:卷的删除要靠 EBS CSI 插件执行。

## 11. 参考文档

- [ClickHouse Docker 安装(含 arm64 / Graviton 兼容性说明)](https://clickhouse.com/docs/get-started/setup/self-managed/docker)
- [ClickHouse Cloud 使用 Graviton 的实践(官方博客)](https://clickhouse.com/blog/graviton-boosts-clickhouse-cloud-performance)
- [Altinity operator 仓库](https://github.com/Altinity/clickhouse-operator) 和 [文档站](https://docs.altinity.com/altinitykubernetesoperator/)
- [ClickHouse 官方 operator 文档](https://clickhouse.com/docs/products/kubernetes-operator/overview)
