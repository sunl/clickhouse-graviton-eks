# 在 EKS 上用 Graviton 节点部署 ClickHouse

本文档是一份可以从头照着执行的部署手册:在一个已有的 EKS 集群里,用 Graviton(arm64)节点部署一套带持久化存储和高可用的 ClickHouse 集群。

部署方式有两种,任选其一:用 operator 管理,或者不用 operator、直接用 StatefulSet。两种方式的准备工作、节点组和存储是共用的。

文档里的命令和清单都在一个 EKS 1.37 集群上按本文的顺序实际执行过,例外情况列在第 10 节。

## 1. 概览

部署完成后的样子:

| 组成 | 内容 |
|---|---|
| 节点 | 一个专用节点组,4 台 c8g.8xlarge(32 vCPU / 64 GB),带污点,只跑 ClickHouse 相关的 pod |
| ClickHouse | 2 分片 × 2 副本,每个 pod 独占一台节点,申请 28 vCPU / 48Gi 内存,数据卷 500Gi gp3 |
| Keeper | 3 副本,分在 3 台节点上,数据卷 20Gi gp3 |
| 访问方式 | 集群内通过 Service 访问,应用使用单独的账号 `app` |

测试用的版本:

| 组件 | 版本 |
|---|---|
| Kubernetes | 1.37(服务端 v1.37.1,节点 kubelet v1.37.1,AL2023 arm64) |
| ClickHouse / Keeper | `clickhouse/clickhouse-server:26.10`、`clickhouse/clickhouse-keeper:26.10`(实际版本 26.10.6.18) |
| Altinity clickhouse-operator(方式一) | 0.27.4 |
| EBS CSI 插件 | v1.68.0-eksbuild.1 |
| eks-pod-identity-agent | v1.4.1-eksbuild.3 |
| 客户端工具 | AWS CLI 2.36.47,kubectl v1.37.2,helm v3.19.0 |

规格和副本数是按 c8g.8xlarge 写的示例值,请按自己的数据量和负载调整。换机型时,同步修改清单里的 CPU、内存和卷大小,让每个 pod 的申请量小于单台节点的可分配资源。

ClickHouse 的镜像标签也可以换成 LTS 版本线;本文只测试了 26.10。

## 2. 准备工作

### 2.1 工具和清单

需要 AWS CLI v2、kubectl,以及本仓库的 `manifests/` 目录;方式一还需要 helm 3。下面所有命令都在仓库根目录执行。

### 2.2 前提条件

- 一个已有的 EKS 集群。方式一要求 Kubernetes 1.25 以上(Altinity operator 的要求)。
- 集群里已经有不带污点的节点。本文创建的 ClickHouse 节点组带污点,CoreDNS、EBS CSI 控制器这些系统组件和业务应用都不会被调度上去,它们需要跑在集群的其他节点上。测试集群里它们跑在另一个不带污点的 arm64 节点组上。
- 集群所在的子网能访问公网(公有子网,或私有子网加 NAT),否则节点拉不了镜像。
- 一个节点 IAM 角色,挂有 `AmazonEKSWorkerNodePolicy`、`AmazonEKS_CNI_Policy`、`AmazonEC2ContainerRegistryReadOnly`。集群已有节点组的话可以直接沿用它的角色;没有的话参考 [创建 EKS 节点 IAM 角色](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html)。
- 实例类型要用 Graviton 2 及以上。ClickHouse 官方 arm64 镜像要求 ARMv8.2-A 加 RCpc 扩展,Graviton 1(A1 实例)不能用。

### 2.3 设置变量

后面的命令都引用这几个变量,先按自己的环境设置好:

```bash
export REGION=us-east-1
export CLUSTER=eks-test
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export NODE_ROLE_ARN=arn:aws:iam::${ACCOUNT_ID}:role/<节点角色名>
export SUBNETS="<subnet-1> <subnet-2> <subnet-3>"
export EBS_CSI_ROLE=AmazonEKSPodIdentityAmazonEBSCSIRole
```

子网和节点角色可以这样查:

```bash
# 集群使用的子网
aws eks describe-cluster --region "$REGION" --name "$CLUSTER" \
  --query 'cluster.resourcesVpcConfig.subnetIds' --output text

# 已有节点组使用的节点角色
aws eks describe-nodegroup --region "$REGION" --cluster-name "$CLUSTER" \
  --nodegroup-name <已有节点组名> --query 'nodegroup.nodeRole' --output text
```

子网建议覆盖 3 个可用区。

### 2.4 连接集群

```bash
aws eks update-kubeconfig --region "$REGION" --name "$CLUSTER"
kubectl get nodes
```

如果 kubectl 返回 `Unauthorized`,说明当前的 IAM 身份不在集群的 access entries 里。创建集群的身份自动有权限,其他身份需要由管理员添加:

```bash
aws eks create-access-entry --region "$REGION" --cluster-name "$CLUSTER" \
  --principal-arn <你的 IAM 角色 ARN>

aws eks associate-access-policy --region "$REGION" --cluster-name "$CLUSTER" \
  --principal-arn <你的 IAM 角色 ARN> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

## 3. 创建专用节点组

```bash
aws eks create-nodegroup --region "$REGION" --cluster-name "$CLUSTER" \
  --nodegroup-name clickhouse \
  --instance-types c8g.8xlarge \
  --ami-type AL2023_ARM_64_STANDARD \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=4,maxSize=4,desiredSize=4 \
  --disk-size 100 \
  --subnets $SUBNETS \
  --node-role "$NODE_ROLE_ARN" \
  --labels workload=clickhouse \
  --taints key=dedicated,value=clickhouse,effect=NO_SCHEDULE

aws eks wait nodegroup-active --region "$REGION" --cluster-name "$CLUSTER" --nodegroup-name clickhouse
```

要点:

- `--ami-type` 必须选 ARM 的 AMI 类型(`AL2023_ARM_64_STANDARD`)。
- 污点 `dedicated=clickhouse:NoSchedule` 把其他工作负载挡在这组节点之外;标签 `workload=clickhouse` 用来把 ClickHouse 限定在这组节点里。两者要一起用:只有污点的话,ClickHouse 的 pod 仍可能被调度到集群里其他的 arm64 节点上。
- 节点数至少 4 台:清单要求每个 ClickHouse pod 独占一台节点,2 分片 × 2 副本就是 4 台。Keeper 与 ClickHouse 共用这些节点。

确认节点:

```bash
kubectl get nodes -l workload=clickhouse -L node.kubernetes.io/instance-type,topology.kubernetes.io/zone
```

应该看到 4 台 `Ready` 的 c8g.8xlarge 节点。再检查这些节点上的系统 pod 是否都正常:

```bash
kubectl get pods -A -o wide | grep -E "$(kubectl get nodes -l workload=clickhouse -o name | sed 's#node/##' | paste -sd'|')"
```

所有 pod 都应该是 `Running`。如果某个 DaemonSet 的 pod 一直 `CrashLoopBackOff`,通常是它的镜像没有 arm64 版本,需要先升级对应的插件。

## 4. 安装存储插件

ClickHouse 和 Keeper 的数据卷用 EBS gp3。要让卷能建出来,集群需要 EBS CSI 和 Pod Identity Agent 两个插件;新建的 EKS 集群默认两个都不带,没有 EBS CSI 时 PVC 会一直 Pending。

先看集群里已经装了哪些:

```bash
aws eks list-addons --region "$REGION" --cluster-name "$CLUSTER"
```

已经有的跳过对应步骤。

**1. 给 EBS CSI 建 IAM 角色**

```bash
aws iam create-role --role-name "$EBS_CSI_ROLE" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

aws iam attach-role-policy --role-name "$EBS_CSI_ROLE" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

这个角色不绑定具体集群,同一账号下的多个集群可以共用。

**2. 安装 Pod Identity Agent**

```bash
aws eks create-addon --region "$REGION" --cluster-name "$CLUSTER" \
  --addon-name eks-pod-identity-agent
```

**3. 安装 EBS CSI 并关联角色**

```bash
aws eks create-addon --region "$REGION" --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver \
  --pod-identity-associations "serviceAccount=ebs-csi-controller-sa,roleArn=arn:aws:iam::${ACCOUNT_ID}:role/${EBS_CSI_ROLE}"

aws eks wait addon-active --region "$REGION" --cluster-name "$CLUSTER" --addon-name aws-ebs-csi-driver
```

**4. 创建 gp3 StorageClass**

```bash
kubectl apply -f manifests/gp3-storageclass.yaml
```

清单里每个数据卷都显式写了 `storageClassName: gp3`,不依赖集群的默认 StorageClass,所以不需要改动集群的默认设置。集群自带的 `gp2` StorageClass 本文没有使用。

卷用的是 gp3 的默认性能(3000 IOPS、125 MB/s)。需要更高性能时,在 `manifests/gp3-storageclass.yaml` 的 `parameters` 里设置 `iops` 和 `throughput`。

## 5. 选择部署方式

两种方式部署出来的集群拓扑、规格、存储都一样,区别在于谁来管理它:

| | 方式一:operator | 方式二:StatefulSet |
|---|---|---|
| 集群里多出来的东西 | 一个 operator 和 4 个 CRD | 没有,只有 StatefulSet、ConfigMap、Service、Secret |
| 拓扑和配置写在哪 | 一个自定义资源里,operator 据此生成配置 | 自己维护的 ConfigMap 里 |
| 部署耗时(实测) | Keeper 82 秒,ClickHouse 131 秒 | Keeper 20 秒,ClickHouse 29 秒 |
| 部署后拓扑生效 | 要等配置同步到各 pod,期间不能执行 `ON CLUSTER` 语句 | 配置是静态的,启动即生效 |
| 应用用的 Service | headless,由客户端自己选副本 | ClusterIP,连接被分摊到各副本 |
| 删除集群资源时的数据卷 | operator 把 PVC 和 EBS 卷一起删掉 | PVC 保留,需要手动删 |
| 适合 | 分片副本多、经常调整拓扑 | 拓扑固定、不想引入额外组件 |

方式一见第 6 节,方式二见第 7 节。两种方式用的命名空间、Secret 名称相同,不要在同一个集群里同时部署。

## 6. 方式一:用 operator 部署

### 6.1 安装 operator

```bash
helm repo add altinity https://docs.altinity.com/clickhouse-operator/
helm repo update

helm install clickhouse-operator altinity/altinity-clickhouse-operator --version 0.27.4 \
  -n clickhouse-operator --create-namespace \
  -f manifests/operator-values.yaml \
  --wait --timeout 5m
```

`manifests/operator-values.yaml` 做了两件事:

- 给 operator 加上节点选择和容忍,让它也跑在专用节点组上。
- 设置 `watchNamespaces: [clickhouse]`。operator 默认只监听自己所在的命名空间;不设置的话,在别的命名空间里创建 ClickHouse 集群不会有任何反应,也没有报错。

验证:

```bash
kubectl -n clickhouse-operator get pods      # 应为 2/2 Running
kubectl get crd | grep altinity.com          # 应有 4 个 CRD
```

### 6.2 可选:先做一次快速验证

如果想在上生产规格之前,先确认镜像、调度和 operator 都工作正常,可以先部署一个小规格的临时集群(1 副本 Keeper 加 1 分片 2 副本 ClickHouse,数据在容器本地盘):

```bash
kubectl create namespace clickhouse
kubectl apply -f manifests/quick-test.yaml

kubectl -n clickhouse wait --for=jsonpath='{.status.status}'=Completed \
  clickhousekeeperinstallations/quick-keeper --timeout=10m
kubectl -n clickhouse wait --for=jsonpath='{.status.status}'=Completed \
  clickhouseinstallations/quick --timeout=10m
```

实测约 72 秒就绪。确认它跑在 arm64 上:

```bash
kubectl -n clickhouse exec chi-quick-test-0-0-0 -c clickhouse -- \
  clickhouse-client -q "SELECT version(), (SELECT value FROM system.build_options WHERE name='SYSTEM_PROCESSOR')"
```

输出应为 `26.10.6.18  aarch64`。

验证完删除:

```bash
kubectl -n clickhouse delete clickhouseinstallations quick
kubectl -n clickhouse delete clickhousekeeperinstallations quick-keeper
```

这个清单把 Keeper 和 ClickHouse 放在一个文件里一起部署,ClickHouse 启动时会报几条连不上 Keeper 的错误,Keeper 就绪后自动恢复。生产部署要分两步,见下一节。

### 6.3 部署 Keeper 和 ClickHouse

**1. 创建命名空间**(做过 6.2 的话已经有了)

```bash
kubectl create namespace clickhouse
```

**2. 创建应用账号的密码**

```bash
kubectl -n clickhouse create secret generic clickhouse-app-user \
  --from-literal=password="$(openssl rand -base64 24)"
```

**3. 把应用账号的来源网段改成自己的 VPC 网段**

`manifests/prod-clickhouse.yaml` 里的 `app/networks/ip` 限定了 `app` 账号可以从哪些地址登录,清单里写的是测试环境的网段 `10.0.0.0/20`。查自己 VPC 的网段并替换:

```bash
VPC_ID=$(aws eks describe-cluster --region "$REGION" --name "$CLUSTER" \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)
aws ec2 describe-vpcs --region "$REGION" --vpc-ids "$VPC_ID" --query 'Vpcs[0].CidrBlock' --output text
```

**4. 部署 Keeper,等它就绪**

```bash
kubectl apply -f manifests/prod-keeper.yaml
kubectl -n clickhouse wait --for=jsonpath='{.status.status}'=Completed \
  clickhousekeeperinstallations/prod-keeper --timeout=10m
```

**5. 部署 ClickHouse,等它就绪**

```bash
kubectl apply -f manifests/prod-clickhouse.yaml
kubectl -n clickhouse wait --for=jsonpath='{.status.status}'=Completed \
  clickhouseinstallations/prod --timeout=15m
```

一定要等 Keeper 就绪后再部署 ClickHouse。按这个顺序,ClickHouse pod 的启动日志里不会有 Keeper 相关的报错。

实测 Keeper 82 秒就绪,ClickHouse 131 秒就绪,pod 没有重启。

**6. 看结果**

```bash
kubectl -n clickhouse get pods -o wide
kubectl -n clickhouse get pvc
```

应该看到 4 个 `chi-prod-main-*` 和 3 个 `chk-prod-keeper-*` 的 pod 都是 `1/1 Running`,4 个 ClickHouse pod 分在 4 台不同的节点上,7 个 PVC 都是 `Bound`。

### 6.4 验证

下面的命令都通过第一个 ClickHouse pod 执行,先定义一个简写:

```bash
CH="kubectl -n clickhouse exec chi-prod-main-0-0-0 -c clickhouse -- clickhouse-client"
```

**版本和架构**

```bash
$CH -q "SELECT version(), (SELECT value FROM system.build_options WHERE name='SYSTEM_PROCESSOR')"
```

输出应为 `26.10.6.18  aarch64`。

**集群拓扑**

```bash
$CH -q "SELECT hostName() AS host, count() AS hosts_seen FROM clusterAllReplicas(main, system.clusters) WHERE cluster='main' GROUP BY host ORDER BY host"
```

应该有 4 行,每行的 `hosts_seen` 都是 4,表示每个副本都看到了完整的拓扑。

**如果行数或数字不对,等一两分钟再查。** operator 状态显示 Completed 之后,新的配置还要过一会儿才同步到每个 pod 里。在拓扑同步完成之前执行 `ON CLUSTER` 的建表语句,只会落到部分副本上。

**复制和分布式表**

建一张复制表和一张分布式表,写入 100 万行,检查每个副本的行数,然后删掉:

```bash
$CH -q "CREATE TABLE default.verify_local ON CLUSTER main (id UInt64, v Float64) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/verify_local', '{replica}') ORDER BY id"
$CH -q "CREATE TABLE default.verify ON CLUSTER main AS default.verify_local ENGINE = Distributed(main, default, verify_local, cityHash64(id))"
$CH -q "INSERT INTO default.verify SELECT number, rand() FROM numbers(1000000) SETTINGS distributed_foreground_insert = 1"
$CH -q "SYSTEM SYNC REPLICA ON CLUSTER main default.verify_local"

$CH -q "SELECT hostName() AS host, count() AS rows FROM clusterAllReplicas(main, default.verify_local) GROUP BY host ORDER BY host"
$CH -q "SELECT count() FROM default.verify"
$CH -q "SELECT hostName() AS host, is_readonly, queue_size, active_replicas, total_replicas FROM clusterAllReplicas(main, system.replicas) WHERE table = 'verify_local' ORDER BY host"

$CH -q "DROP TABLE default.verify ON CLUSTER main SYNC"
$CH -q "DROP TABLE default.verify_local ON CLUSTER main SYNC"
```

实测结果:

- 同一个分片的两个副本行数相同(分片 1 各 499,487 行,分片 2 各 500,513 行)。
- 分布式表的总行数是 1,000,000。
- 4 个副本都是 `is_readonly=0`、`queue_size=0`、`active_replicas=2`、`total_replicas=2`。

**数据卷**

```bash
kubectl -n clickhouse exec chi-prod-main-0-0-0 -c clickhouse -- df -h /var/lib/clickhouse
```

应该看到一块约 492G 的盘挂在 `/var/lib/clickhouse`,这就是 gp3 数据卷。

### 6.5 应用如何连接

| 项 | 值 |
|---|---|
| 地址 | `clickhouse-prod.clickhouse.svc.cluster.local`(同命名空间内可简写为 `clickhouse-prod`) |
| 端口 | 9000(原生协议),8123(HTTP) |
| 账号 | `app` |
| 密码 | Secret `clickhouse-app-user` 的 `password` 字段 |

这个 Service 是 headless 的:域名解析到全部 4 个副本的 IP,由客户端自己选一个连接,Kubernetes 不在中间做负载均衡。连到任意一个副本都可以查询分布式表。要让读写均匀分布在各副本上,需要客户端自己轮换地址,或者在清单里另外定义一个带负载均衡的 Service。

**不要让应用使用 `default` 账号。** operator 把 `default` 账号限制为只允许 ClickHouse 自己的 pod 使用,从其他 pod 用它连接会返回 `AUTHENTICATION_FAILED`。应用要用清单里定义的 `app` 账号,或者按同样的写法在 `manifests/prod-clickhouse.yaml` 的 `configuration.users` 里增加账号。

应用的 pod 建议用环境变量从 Secret 读取密码:

```yaml
env:
  - name: CLICKHOUSE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: clickhouse-app-user
        key: password
```

Secret 在 `clickhouse` 命名空间里,其他命名空间的应用需要在自己的命名空间里也建一份。

**连接测试。** `manifests/client-test.yaml` 是一个模拟应用的 pod,用 `app` 账号分别通过原生协议和 HTTP 各查询一次:

```bash
kubectl apply -f manifests/client-test.yaml
kubectl -n clickhouse wait --for=jsonpath='{.status.phase}'=Succeeded pod/ch-client-test --timeout=180s

kubectl -n clickhouse logs ch-client-test -c native    # 输出:账号、版本、可达的副本数
kubectl -n clickhouse logs ch-client-test -c http      # 输出:账号、响应请求的副本

kubectl -n clickhouse delete pod ch-client-test
```

实测输出:

```
app	26.10.6.18	4
app	chi-prod-main-0-1-0
```

这个测试 pod 没有加容忍,会被调度到集群里的其他节点上,和真实的业务应用一样。

`app` 账号定义在配置文件里,默认拥有全部权限。需要限制权限的话,按 ClickHouse 的用户配置方式收紧。

## 7. 方式二:不用 operator,用 StatefulSet 部署

这种方式只用 Kubernetes 自带的对象。Keeper 和 ClickHouse 各是一个 StatefulSet,配置写在 ConfigMap 里:

| 文件 | 内容 |
|---|---|
| `manifests/statefulset/keeper.yaml` | Keeper 的配置、headless Service、3 副本的 StatefulSet |
| `manifests/statefulset/clickhouse.yaml` | ClickHouse 的集群配置和账号配置、两个 Service、4 个 pod 的 StatefulSet |
| `manifests/statefulset/client-test.yaml` | 连接测试用的 pod |

### 7.1 清单的几个设计点

- **分片和副本的对应关系是按 pod 序号算的**:`clickhouse-0`、`clickhouse-1` 是分片 1,`clickhouse-2`、`clickhouse-3` 是分片 2。启动命令里用 `序号 / 2 + 1` 算出分片号,写进 `{shard}` 宏;`{replica}` 宏就是 pod 名。集群拓扑(`remote_servers`)在 ConfigMap 里按 pod 的固定域名逐个列出。要改每个分片的副本数或分片数,启动命令里的算式、`remote_servers` 和 StatefulSet 的 `replicas` 三处要一起改。
- **节点之间的认证用集群密钥**:`remote_servers` 里配置了 `<secret>`,值来自 Secret `clickhouse-cluster-secret`。节点之间执行分布式查询时用它互相认证,不需要在配置里写账号密码。
- **`default` 账号只能在 pod 本机使用**:用于 `kubectl exec` 进去做管理操作。应用使用 `app` 账号。
- **进程以镜像里的 `clickhouse` 用户(uid 101)运行**,不是 root。清单直接启动 `clickhouse-server` 和 `clickhouse-keeper`,没有经过镜像自带的入口脚本,所以账号都以清单里的配置为准。
- **两个 Service**:`clickhouse-headless` 给副本之间互相访问用,每个 pod 一个固定域名;`clickhouse` 是普通的 ClusterIP Service,给应用用。

### 7.2 部署

**1. 创建命名空间和两个 Secret**

```bash
kubectl create namespace clickhouse

# 应用账号的密码
kubectl -n clickhouse create secret generic clickhouse-app-user \
  --from-literal=password="$(openssl rand -base64 24)"

# 节点之间互相认证用的集群密钥
kubectl -n clickhouse create secret generic clickhouse-cluster-secret \
  --from-literal=secret="$(openssl rand -hex 32)"
```

**2. 把应用账号的来源网段改成自己的 VPC 网段**

`manifests/statefulset/clickhouse.yaml` 的 `users.xml` 里,`app` 账号的 `<ip>` 写的是测试环境的网段 `10.0.0.0/20`。查自己 VPC 的网段并替换:

```bash
VPC_ID=$(aws eks describe-cluster --region "$REGION" --name "$CLUSTER" \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)
aws ec2 describe-vpcs --region "$REGION" --vpc-ids "$VPC_ID" --query 'Vpcs[0].CidrBlock' --output text
```

**3. 部署 Keeper,等它就绪**

```bash
kubectl apply -f manifests/statefulset/keeper.yaml
kubectl -n clickhouse rollout status statefulset/clickhouse-keeper --timeout=10m
```

**4. 部署 ClickHouse,等它就绪**

```bash
kubectl apply -f manifests/statefulset/clickhouse.yaml
kubectl -n clickhouse rollout status statefulset/clickhouse --timeout=15m
```

实测 Keeper 20 秒就绪,ClickHouse 29 秒就绪,pod 没有重启。

**5. 看结果**

```bash
kubectl -n clickhouse get pods -o wide
kubectl -n clickhouse get pvc
```

应该看到 `clickhouse-0` 到 `clickhouse-3`、`clickhouse-keeper-0` 到 `clickhouse-keeper-2` 共 7 个 pod 都是 `1/1 Running`,4 个 ClickHouse pod 分在 4 台不同的节点上,7 个 PVC 都是 `Bound`。

4 个 ClickHouse pod 是同时启动的,首次部署时某个 pod 启动得早,其他 pod 的域名还解析不到,它的服务日志里会有几条 `Not found address of host` 的错误。pod 都起来之后不再出现,不影响使用。

### 7.3 验证

```bash
CH="kubectl -n clickhouse exec clickhouse-0 -c clickhouse -- clickhouse-client"
```

**版本和架构**

```bash
$CH -q "SELECT version(), (SELECT value FROM system.build_options WHERE name='SYSTEM_PROCESSOR')"
```

输出应为 `26.10.6.18  aarch64`。

**分片和副本的对应关系**

```bash
$CH -q "SELECT hostName() AS host, getMacro('shard') AS shard, getMacro('replica') AS replica FROM clusterAllReplicas(main, system.one) ORDER BY host"
```

实测输出:

```
clickhouse-0	1	clickhouse-0
clickhouse-1	1	clickhouse-1
clickhouse-2	2	clickhouse-2
clickhouse-3	2	clickhouse-3
```

能查出 4 行,同时说明节点之间用集群密钥的互相访问是通的。

**复制和分布式表**

和方式一用同样的语句:

```bash
$CH -q "CREATE TABLE default.verify_local ON CLUSTER main (id UInt64, v Float64) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/verify_local', '{replica}') ORDER BY id"
$CH -q "CREATE TABLE default.verify ON CLUSTER main AS default.verify_local ENGINE = Distributed(main, default, verify_local, cityHash64(id))"
$CH -q "INSERT INTO default.verify SELECT number, rand() FROM numbers(1000000) SETTINGS distributed_foreground_insert = 1"
$CH -q "SYSTEM SYNC REPLICA ON CLUSTER main default.verify_local"

$CH -q "SELECT hostName() AS host, count() AS rows FROM clusterAllReplicas(main, default.verify_local) GROUP BY host ORDER BY host"
$CH -q "SELECT count() FROM default.verify"
$CH -q "SELECT hostName() AS host, is_readonly, queue_size, active_replicas, total_replicas FROM clusterAllReplicas(main, system.replicas) WHERE table = 'verify_local' ORDER BY host"

$CH -q "DROP TABLE default.verify ON CLUSTER main SYNC"
$CH -q "DROP TABLE default.verify_local ON CLUSTER main SYNC"
```

实测结果和方式一相同:`clickhouse-0` 和 `clickhouse-1` 各 499,487 行,`clickhouse-2` 和 `clickhouse-3` 各 500,513 行,分布式表共 1,000,000 行,4 个副本的复制队列都是 0。

**数据卷**

```bash
kubectl -n clickhouse exec clickhouse-0 -c clickhouse -- df -h /var/lib/clickhouse
```

应该看到一块约 492G 的盘挂在 `/var/lib/clickhouse`。

### 7.4 应用如何连接

| 项 | 值 |
|---|---|
| 地址 | `clickhouse.clickhouse.svc.cluster.local`(同命名空间内可简写为 `clickhouse`) |
| 端口 | 9000(原生协议),8123(HTTP) |
| 账号 | `app` |
| 密码 | Secret `clickhouse-app-user` 的 `password` 字段 |

这个 Service 会把连接分摊到就绪的副本上。实测从一个客户端 pod 连续建立 40 个连接,落到 4 个副本上的数量分别是 13、6、11、10。

应用 pod 读取密码的写法和 6.5 相同。要增加账号,在 `manifests/statefulset/clickhouse.yaml` 的 `users.xml` 里照着 `app` 再写一段,然后按 7.5 的方法让配置生效。

**连接测试**

```bash
kubectl apply -f manifests/statefulset/client-test.yaml
kubectl -n clickhouse wait --for=jsonpath='{.status.phase}'=Succeeded pod/ch-client-test --timeout=180s

kubectl -n clickhouse logs ch-client-test -c native    # 输出:账号、版本、可达的副本数
kubectl -n clickhouse logs ch-client-test -c http      # 输出:账号、响应请求的副本

kubectl -n clickhouse delete pod ch-client-test
```

实测输出:

```
app	26.10.6.18	4
app	clickhouse-2
```

用 `default` 账号从其他 pod 连接会返回 `AUTHENTICATION_FAILED`。

### 7.5 修改配置后让它生效

配置在 ConfigMap 里,改完之后要重启 pod 才会重新加载:

```bash
kubectl apply -f manifests/statefulset/clickhouse.yaml
kubectl -n clickhouse rollout restart statefulset/clickhouse
kubectl -n clickhouse rollout status statefulset/clickhouse --timeout=15m
```

重启是逐个 pod 进行的。实测 4 个 pod 全部重启完用了 73 秒,期间始终至少有 3 个副本处于就绪状态,重启后分片和副本的对应关系不变。

只 `apply` 不重启是不生效的。实测在 `users.xml` 里加了一个账号:`apply` 之后等了 90 秒,ClickHouse 里仍然没有这个账号;滚动重启之后,4 个副本上都有了。

## 8. 高可用:实测结果和限制

两种方式都做了同样的故障测试,结果一致:

| 测试 | 结果 |
|---|---|
| 删掉 1 个 Keeper pod,在它完全消失后立即写入 | 写入成功,同分片的另一个副本同步到了数据 |
| 删掉 1 个 ClickHouse 副本 pod | 重建期间,通过其他副本查询分布式表正常 |
| 上面那个 pod 重建之后 | 新 pod 挂回原来的 EBS 卷,该副本原有的 5,001,171 行数据都在 |
| pod 分布 | 4 个 ClickHouse pod 在 4 台不同节点上,3 个 Keeper pod 在 3 台不同节点上 |

两个限制:

- **节点数**:清单用按主机名的 pod 反亲和,每个 ClickHouse pod 独占一台节点,所以节点数不能少于 ClickHouse 的 pod 数。扩分片或副本之前要先扩节点组。
- **不保证跨可用区**:按主机名的反亲和只保证不同机,不保证不同可用区。实测中有一次部署,3 个 Keeper 里有 2 个落在了同一个可用区,这个可用区出问题时 Keeper 会失去多数派。要保证跨可用区分散,需要另外加按可用区的调度约束(如 `topologySpreadConstraints`),并保证每个可用区有足够的节点。

**方式一的误删保护**:删除 `ClickHouseInstallation` 或 `ClickHouseKeeperInstallation` 资源时,operator 会把对应的 PVC 一并删除,EBS 卷和数据也随之删除。要在删除后保留数据,需要在清单的卷模板里设置 `reclaimPolicy: Retain`。

## 9. 需要注意的问题

| 问题 | 现象 | 原因和处理 |
|---|---|---|
| 应用连不上 | 用 `default` 账号从业务 pod 连接返回 `AUTHENTICATION_FAILED` | 两种方式里 `default` 账号都不对外开放。用 `app` 账号 |
| `app` 账号也连不上 | 返回 `AUTHENTICATION_FAILED` | 检查 `app` 账号的来源网段是否已改成自己的 VPC 网段 |
| PVC 一直 Pending | pod 起不来 | 没有 EBS CSI 插件或 Pod Identity Agent,见第 4 节 |
| pod 一直 Pending | 提示资源不足或没有匹配的节点 | 节点数少于 ClickHouse 的 pod 数;或清单里申请的 CPU、内存超过了单台节点的可分配量;或节点组的标签、污点和清单不一致 |
| ClickHouse 启动时报 Keeper 连接错误 | 日志里有连不上 Keeper 的报错 | ClickHouse 比 Keeper 先起来。先部署 Keeper 并等它就绪 |
| kubectl 返回 Unauthorized | | 当前 IAM 身份不在集群的 access entries 里,见 2.4 |
| 系统 pod 在 arm64 节点上 CrashLoopBackOff | 日志里有 `exec format error` | 该插件的镜像没有 arm64 版本,升级插件 |
| 方式一:创建集群后没有任何反应 | `ClickHouseInstallation` 的状态一直为空,没有 pod,也没有报错 | operator 没有监听这个命名空间,检查 `watchNamespaces`,见 6.1 |
| 方式一:`ON CLUSTER` 建表只落到部分副本 | 其他副本上查表报 `UNKNOWN_TABLE` | 拓扑还没同步完就执行了建表语句,见 6.4。用 `CREATE TABLE IF NOT EXISTS ... ON CLUSTER` 补建 |
| 方式二:改了 ConfigMap 不生效 | 新账号或新配置没有出现 | 需要重启 pod,见 7.5 |

方式一里,operator 给 `ClickHouseInstallation` 记录的状态中,每次部署都会有几条 `Update Service ... failed with error: just recreate the service in case of service type change`。Service 实际都创建成功,集群功能不受影响。

## 10. 没有验证的部分

- **性能**:没有做基准测试。
- **跨可用区分散**:没有验证按可用区的调度约束。
- **节点和可用区故障**:只测了删除 pod,没有测节点宕机、可用区不可用、EBS 卷故障。
- **日常运维**:版本升级、扩缩分片和副本、卷扩容、备份恢复、监控告警都没有测。方式二测了通过 ConfigMap 增加账号再滚动重启,确认新账号出现在了 4 个副本上,但没有验证用新账号登录。
- **`reclaimPolicy: Retain`**、gp3 的 `iops` 和 `throughput` 参数、对 `app` 账号的权限限制都没有测。
- **其他 ClickHouse 版本**(包括 LTS 版本线)没有测。
- **方式一里带负载均衡的 Service**没有测,只用了 operator 默认创建的 headless Service。
- **2.4 节添加 access entry 的两条命令**没有用 CLI 执行过,测试时是通过 API 完成的。
- **第 4 节的三组 AWS 命令**不是在全新状态下按顺序执行的,因为测试集群里这些资源已经存在:创建 IAM 角色的命令是换了一个角色名执行的;安装 EBS CSI 的命令是在另一个没有该插件的集群上执行的;安装 Pod Identity Agent 的命令返回的是"已存在"。三者的参数写法都通过了,前两者实际创建成功。
- **第 11 节的清理命令**里,集群内的部分(方式一、方式二、附录、删除 StorageClass)都执行过;删除节点组、插件、IAM 角色的 AWS 命令没有按本文的资源名执行过。

## 11. 清理

卷的删除要靠 EBS CSI 插件执行,所以先删集群内的资源,最后才删节点组和插件。

**方式一的集群内资源**

```bash
# ClickHouse 和 Keeper(operator 会把它们的 PVC 和 EBS 卷一起删掉)
kubectl -n clickhouse delete clickhouseinstallations prod
kubectl -n clickhouse delete clickhousekeeperinstallations prod-keeper

# 确认 PVC 和 PV 都已经没有了
kubectl -n clickhouse get pvc
kubectl get pv

# operator、命名空间、CRD
helm uninstall clickhouse-operator -n clickhouse-operator
kubectl delete namespace clickhouse clickhouse-operator
kubectl get crd -o name | grep -E 'clickhouse(-keeper)?\.altinity\.com' | xargs kubectl delete
```

**方式二的集群内资源**

```bash
kubectl -n clickhouse delete statefulset clickhouse clickhouse-keeper

# StatefulSet 删除后 PVC 会保留,要手动删除,否则 EBS 卷继续计费
kubectl -n clickhouse delete pvc --all
kubectl get pv

kubectl delete namespace clickhouse
```

**共用的资源**

```bash
kubectl delete storageclass gp3

# 节点组
aws eks delete-nodegroup --region "$REGION" --cluster-name "$CLUSTER" --nodegroup-name clickhouse

# 插件和 IAM 角色(集群里没有其他工作负载使用 EBS 卷时)
aws eks delete-addon --region "$REGION" --cluster-name "$CLUSTER" --addon-name aws-ebs-csi-driver
aws eks delete-addon --region "$REGION" --cluster-name "$CLUSTER" --addon-name eks-pod-identity-agent
aws iam detach-role-policy --role-name "$EBS_CSI_ROLE" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
aws iam delete-role --role-name "$EBS_CSI_ROLE"
```

## 附录:ClickHouse 官方 operator

ClickHouse 官方也提供了一个 operator。它在 arm64 上能正常工作,但 API 版本还是 `v1alpha1`,后续可能有不兼容的变更,建议只用于 PoC。它和 Altinity operator 的 CRD 不同组,可以装在同一个集群里。

官方 operator 依赖 cert-manager。下面的命令把 cert-manager、operator 和一个示例集群(1 副本 Keeper 加 1 分片 2 副本 ClickHouse,gp3 数据卷)都部署到同一个专用节点组上,前提是已完成第 3、4 节:

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager --version v1.21.3 \
  -n cert-manager --create-namespace \
  -f manifests/official-operator/cert-manager-values.yaml \
  --wait --timeout 6m

helm install clickhouse-operator oci://ghcr.io/clickhouse/clickhouse-operator-helm --version 0.0.8 \
  -n clickhouse-operator-system --create-namespace \
  -f manifests/official-operator/operator-values.yaml \
  --wait --timeout 6m

kubectl create namespace clickhouse-official
kubectl apply -f manifests/official-operator/cluster.yaml

kubectl -n clickhouse-official wait --for=condition=Ready \
  keeperclusters.clickhouse.com/official-keeper --timeout=10m
kubectl -n clickhouse-official wait --for=condition=Ready \
  clickhouseclusters.clickhouse.com/official --timeout=10m
```

实测结果:

- 约 31 秒就绪,ClickHouse 版本 26.10.6.18,架构 aarch64。
- 在 Replicated 数据库里建表、写入 100 万行,另一个副本读到 100 万行。
- 示例清单把 Keeper 和 ClickHouse 一起部署,两个 ClickHouse pod 各重启了 1 次才起来,原因是启动时 Keeper 还没就绪。
- 示例清单没有设置 `spec.settings.defaultUserPassword`,apply 时会警告 `default` 用户没有密码。

和 Altinity operator 的两点区别:

- **卷的回收**:删除 `ClickHouseCluster` 或 `KeeperCluster` 后,PVC 会保留,需要手动删除,否则 EBS 卷继续计费。
- **资源简称**:两个 operator 都装时,`kubectl get chk` 会同时匹配两边的 Keeper 资源并给出警告,这时用全名 `clickhousekeeperinstallations` 或 `keeperclusters`。

清理:

```bash
kubectl -n clickhouse-official delete clickhouseclusters.clickhouse.com official
kubectl -n clickhouse-official delete keeperclusters.clickhouse.com official-keeper
kubectl -n clickhouse-official delete pvc --all
kubectl delete namespace clickhouse-official
helm uninstall clickhouse-operator -n clickhouse-operator-system
kubectl delete namespace clickhouse-operator-system
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
kubectl get crd -o name | grep -E '\.clickhouse\.com$|cert-manager\.io$' | xargs kubectl delete
```

## 参考文档

- [ClickHouse Docker 安装(含 arm64 / Graviton 兼容性说明)](https://clickhouse.com/docs/get-started/setup/self-managed/docker)
- [ClickHouse Cloud 使用 Graviton 的实践(官方博客)](https://clickhouse.com/blog/graviton-boosts-clickhouse-cloud-performance)
- [Altinity operator 仓库](https://github.com/Altinity/clickhouse-operator) 和 [文档站](https://docs.altinity.com/altinitykubernetesoperator/)
- [ClickHouse 官方 operator 文档](https://clickhouse.com/docs/products/kubernetes-operator/overview)
