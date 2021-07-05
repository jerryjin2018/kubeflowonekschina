## 1.介绍
下面是一段Kubeflow官方自己的介绍 (这里是Kubeflow v1.2版本的，Kubeflow最新的版本是v1.3）

```
Kubeflow项目致力于使机器学习（ML）工作流在Kubernetes上的部署变得简单、可移植和可扩展。
我们的目标不是重新创建其他服务，而是提供一种直接的方式，将ML的最佳开源系统部署到不同的基础设施。
在你运行Kubernetes的任何地方，你都应该能够运行Kubeflow。
```
[https://v1-2-branch.kubeflow.org/#overview](https://v1-2-branch.kubeflow.org/#overview)  
简单而言Kubefow就是基于现有的组件，特别是Kubernetes上已有的组件而完成的一个ML的平台。贯穿ML的整个生命周期.


## 2.安装配置

#### 1.创建EKS集群
根据如下的Kubeflow的官方的安装文档，我们可以知道EKS和Kubeflow的兼容性列表(support matrix)  
[https://v1-2-branch.kubeflow.org/docs/aws/deploy/install-kubeflow/](https://v1-2-branch.kubeflow.org/docs/aws/deploy/install-kubeflow/)

EKS兼容性: 从Kubeflow 1.2开始启用了EKS版本和Kubeflow版本之间的E2E测试。

| EKS Versions	  | Kubeflow 1.2 |
|:----------|:----------|
| 1.15	    | compatible    |
| 1.16	    | compatible    |
| 1.17	    | compatible    |
| 1.18	   | compatible    |

**incompatible**: 二者不兼容，无法正常工作。  
**compatible**: 所有的Kubeflow的功能都已在此EKS版本中进行了测试和验证。  
**no known issues**: 该组合尚未经过全面测试，但没有报告说有问题。

这里创建最新的EKS 1.18的集群
> 这里所有的操作都是在 AMZN Linux 2 (RHEL/CentOS系试用，其他Linux发行版需要酌情更改)  
> 根据如下内容创建文件名为 KubeflowOnEKS.yaml 的yaml文件

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: KubeflowOnEKS
  region: cn-northwest-1
  version: "1.18"

managedNodeGroups:
  - name: ng-kubeflow-01
    instanceType: t3.medium
    instanceName: ng-kubeflow-01
    desiredCapacity: 4
    minSize: 1
    maxSize: 6 
    volumeSize: 100
    ssh:
      publicKeyName: <your-key-name>
      allow: true
      enableSsm: true 
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true 
```

> 使用如下的命令创建EKS集群
```
eksctl create cluster -f KubeflowOnEKS.yaml
```

#### <font color=red>Kubeflow官方也强调需要将如下的组件安装好</font>
另外，这里省略了准备环境的过程，比如<font color=red>aws cli, eksctl, kubectl, aws-iam-authenticator</font>等，具体的步骤可以参考我写的另外的EKS workshop for china的步骤。  
[https://github.com/jerryjin2018/AWS-China-EKS-Workshop-2021/blob/main/Lab1:%20Create%20an%20EKS%20cluster%20through%20eksctl.md](https://github.com/jerryjin2018/AWS-China-EKS-Workshop-2021/blob/main/Lab1:%20Create%20an%20EKS%20cluster%20through%20eksctl.md)  


#### 2.准备
在global，使用如下的几步基本上就可以安装好Kubeflow，是相当的简单  

1)列出这里变量的信息,后续安装步骤需要使用
```
aws eks list-clusters --output text | awk '{print "Cluster: "$2}'
aws eks describe-cluster --name $(aws eks list-clusters --output text| awk '{print $2}') | jq .cluster.endpoint | awk -F. '{print "Region: " $3}'
```
输出:
```
[ec2-user@ip-172-31-1-111 kubeflow2021]$ aws eks list-clusters --output text | awk '{print "Cluster: "$2}'
Cluster: KubeflowOnEKS

[ec2-user@ip-172-31-1-111 kubeflow2021]$ aws eks describe-cluster --name $(aws eks list-clusters --output text| awk '{print $2}') | jq .cluster.endpoint | awk -F. '{print "Region: " $3}'
Region: cn-northwest-1
```
2)现在Kubeflow的安装文件(kfctl -- Kubeflow命令行工具)
```
curl -OL https://github.com/kubeflow/kfctl/releases/download/v1.2.0/kfctl_v1.2.0-0-gbc038f9_linux.tar.gz
```
3)解压下载的Kubeflow的安装文件
```
tar -xvf kfctl_v1.2.0*.tar.gz
```  
4)创建环境变量,其中<font color=red>${aws_cluster_name}</font> - 你的eks集群的名称。kfctl会用到这个参数来指定metadata.name.和alb-ingress-controller，一定要正确设置。  
命令设置环境变量：
```
export PATH=$PATH:$(pwd)
export AWS_CLUSTER_NAME=$(aws eks list-clusters --output text | awk '{print $2}')
export AWS_REGION=`aws eks describe-cluster --name $(aws eks list-clusters --output text| awk '{print $2}') | jq .cluster.endpoint | awk -F. '{print $3}'`
```

手动设置环境变量：
```
export PATH=$PATH:"<path to kfctl>"
export AWS_CLUSTER_NAME=<YOUR EKS CLUSTER NAME>
export AWS_REGION=<AWS REGION NAME>
```
设置的为:
```
export PATH=$PATH:"/home/ec2-user/kubeflow2021/"
export AWS_CLUSTER_NAME=KubeflowOnEKS
export AWS_REGION=cn-northwest-1
```
5)创建安装目录和下载kfctl的配置文件，<font color=red>kfctl配置文件(kfctl_aws.v1.2.0.yaml)下载可能有困难,需要科学解决</font>
```
mkdir ${AWS_CLUSTER_NAME} && cd ${AWS_CLUSTER_NAME}
curl -L https://raw.githubusercontent.com/kubeflow/manifests/v1.2-branch/kfdef/kfctl_aws.v1.2.0.yaml -o kfctl_aws.yaml
```
6)修改kfctl的配置文件 
使用<font color=red>(IRSA)</font>  

Kubeflow从v1.0.1版本开始，支持使用AWS IAM Roles for Service Account<font color=red>(IRSA)</font>来精细控制AWS服务的访问。kfctl将创建两个IAM角色 <font color=red>kf-admin-${region}-${cluster_name}</font> 和 <font color=blue>kf-user-${region}-${cluster_name}</font> 以及在 kubeflow 命名空间下创建两个Kubernetes Service Account <font color=red>kf-admin</font> 和 <font color=blue>kf-user</font>。<font color=red>kf-admin-${region}-${cluster_name}</font>将由alb-ingress-controller、profile-controller或任何需要与AWS服务对话的Kubeflow控制面组件使用，<font color=blue>kf-user-${region}-${cluster_name}</font>可由用户的应用程序使用。

AWS IAM Roles for Service Account<font color=red>(IRSA)</font>官方介绍:  
[https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

更改kfctl_aws.yaml中的region和enablePodIamPolicy
```
sed -i "/region:/s/\(^[ \t]*\)\(region: \)\(.*$\)/\1\2${AWS_REGION}/" kfctl_aws.yaml
sed -i "/enablePodIamPolicy:/s/\(^[ \t]*\)\(enablePodIamPolicy: \)\(.*$\)/\1\2true/" kfctl_aws.yaml 
```

#### 3.安装Kubeflow
```
kfctl apply -V -f kfctl_aws.yaml
```

## 3.问题处理
前面谈到在global部署很方便，因为众所周知的FW，所以是无法直接访问存放在gcr(Google Cloud Registry)的容器的image，所以我做了如下的工作：
```
1.找到所以kubeflow在安装过程中需要使用的容器镜像
2.因为NWCD在China维护了一个从global周期性同步容器镜像的repository，所以我给他们提了PR，将这150多个容器的image，同步到China
3.部署mutating admission webhook自动替换需要访问gcr的请求到NWCD的repository
```
1)下载Kubeflow完整的manifest,此问题是使用kustomize打包的
```
curl -OL https://github.com/kubeflow/manifests/archive/v1.2.0.tar.gz
```
2)解压此manifest
```
tar zxvf v1.2.0.tar.gz
```
3)查找容器的image主要来自于那些repository
```
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -n "image: " | wc -l
892
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -n "image: gcr.io" | wc -l
476
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -n "image: " | grep -v "image: gcr.io" | wc -l
416
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -n "image: " | awk -Fimage: '{print $2}' | awk -F/ '{print $1}' | sort | uniq -c | sort -k 1 -nr
    476  gcr.io
    190  docker.io
     29  quay.io
     24  argoproj
     20  "docker.io
     12  'gcr.io
     11  seldonio
     10  "{{ annotation .ObjectMeta `sidecar.istio.io
      9  mysql:8.0.3
      9  mysql:8
      8  python:3.7
      8  mysql:5.6
      7  mpioperator
      6  [[ annotation .ObjectMeta `sidecar.istio.io
      5  "{{ .Values.global.proxy_init.image }}"
      5  {{ $.Values.global.proxy.enableCoreDumpImage }}
      5  "{{ .Values.global.hub }}
      5  grafana
      5  \"docker.io
      4  vertaaiofficial
      4  busybox
      3  "quay.io
      3  minio
      3  kubeflow
      3  amazon
      2  $(tekton-registry)
      2  python:alpine3.6
      2  osixia
      2  nvidia
      2  nvcr.io
      2  metacontroller
      2  cos-nvidia-installer:fixed
      2  aipipeline
      1  seedjeffwan
      1  registry.redhat.io
      1  $(registry)
      1  mysql:5.7
      1  mxjob
      1  "mintel
      1  "minio
      1  keycloak
      1  "grafana
      1  fluent
      1  'docker.io
      1  busybox:latest
```
3)针对如上3个主要容器的image的repository进行筛选
```
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: gcr.io" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u | wc -l
112
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: gcr.io" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u | more
image: gcr.io/arrikto/kubeflow/oidc-authservice:28c59ef
image: gcr.io/arrikto/kubeflow/oidc-authservice:6ac9400
image: gcr.io/cloud-solutions-group/cloud-endpoints-controller:0.2.1
image: gcr.io/cloud-solutions-group/esp-sample-app:1.0.0
...
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: docker.io" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u | wc -l
28
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: docker.io" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u | more
image: docker.io/aipipeline/api-server:0.4.0
image: docker.io/aipipeline/frontend:0.4.0
image: docker.io/aipipeline/metadata-writer:0.4.0
image: docker.io/aipipeline/persistenceagent:0.4.0
...
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: quay.io" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u | wc -l
9
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: quay.io" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u | more
image: quay.io/dexidp/dex:v2.22.0
image: quay.io/jetstack/cert-manager-cainjector:v0.11.0
image: quay.io/jetstack/cert-manager-controller:v0.11.0
image: quay.io/jetstack/cert-manager-webhook:v0.11.0
...
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: argoproj" | sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u
image: argoproj/argoui:v2.3.0
image: argoproj/workflow-controller:v2.3.0
[ec2-user@ip-172-31-1-73 kubeflow2021]$ find ./manifests-1.2.0/ -iname "*.yaml" | xargs  grep -h "image: seldonio"| sed 's/^[ \t]*//;s/^-[ \t]*//' | sort -u
image: seldonio/alibiexplainer:1.4.0
image: seldonio/mlflowserver_grpc
image: seldonio/mlflowserver_rest
image: seldonio/mlserver
image: seldonio/sklearnserver_grpc
image: seldonio/sklearnserver_rest
image: seldonio/tfserving-proxy_grpc
image: seldonio/tfserving-proxy_rest
image: seldonio/xgboostserver_grpc
image: seldonio/xgboostserver_rest
```

4)提供这些image给NWCD，具体的PR如下  
[https://github.com/jerryjin2018/container-mirror/blob/master/mirror/required-images.txt](https://github.com/jerryjin2018/container-mirror/blob/master/mirror/required-images.txt)

5)解决容器的image的问题后，部署mutating admission webhook自动替换需要访问gcr的请求到NWCD的repository. 
[https://github.com/nwcdlabs/container-mirror/blob/master/webhook/README.md](https://github.com/nwcdlabs/container-mirror/blob/master/webhook/README.md)
```
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml
```
6)至此解决了主要的容器的image的问题

## 4.从Kubeflow社区上游彻底解决问题
2020年12月29日给Kubeflow社区提了如下issue  
[https://github.com/kubeflow/kubeflow/issues/5487](https://github.com/kubeflow/kubeflow/issues/5487)
```json
/kind bug
current kfctl_aws(kfctl_aws.v1.2.0.yaml) can't to deploy kubeflow in AWS China

What steps did you take and what happened:

I downloaded the latest yaml file for kfctl_aws from official site ( https://raw.githubusercontent.com/kubeflow/manifests/v1.2-branch/kfdef/kfctl_aws.v1.2.0.yaml )

I built my EKS cluster in AWS China ( Ningxia ) region through eksctl succesfully

I tried the steps in the link ( https://www.kubeflow.org/docs/aws/deploy/install-kubeflow/ ), but can't deploy the Kubeflow successfully.

The main issue is "ImagePullBackOff" after I checked the status of pods through command ( kubectl get po -n namespaces)
The reason behind this is that the hosts in China mainland can't to pull the images from gcr.io site.

I modify almost all the yaml files inside https://github.com/kubeflow/manifests/archive/v1.2.0.tar.gz with images migrated from gcr.io.

I also found that the issue with difference between AWS global and AWS China -- In the Beijing and Ningxia Regions, the Amazon Resource Name (ARN) syntax includes a cn. -- arn:aws-cn in AWS China rather than arn:aws( https://docs.amazonaws.cn/en_us/aws/latest/userguide/ARNs.html )

After changed and troubleshooted many points, I can get Kubeflow the web portal in AWS China region.

What did you expect to happen:

I expect there is general common for assign images repositories location. such as: gcr.io and docker, even others through environment variables.
To change aws arn from aws to aws-cn for AWS China region's users.
Anything else you would like to add:
[Miscellaneous information that will assist in solving the issue.]

Environment:

Kubeflow version: (version number can be found at the bottom left corner of the Kubeflow dashboard): 1.2 with build version v1beta1

kfctl version: (use kfctl version): kfctl v1.2.0-0-gbc038f9

Kubernetes platform: (e.g. minikube) 1.18

Kubernetes version: (use kubectl version):

Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.4", GitCommit:"d360454c9bcd1634cf4cc52d1867af5491dc9c5f", GitTreeState:
"clean", BuildDate:"2020-11-11T13:17:17Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18+", GitVersion:"v1.18.9-eks-d1db3c", GitCommit:"d1db3c46e55f95d6a7d3e5578689371318f95ff9", G
itTreeState:"clean", BuildDate:"2020-10-20T22:18:07Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}

OS (e.g. from /etc/os-release):
NAME="Amazon Linux"
VERSION="2"
ID="amzn"
ID_LIKE="centos rhel fedora"
VERSION_ID="2"
PRETTY_NAME="Amazon Linux 2"
ANSI_COLOR="0;33"
CPE_NAME="cpe:2.3⭕amazon:amazon_linux:2"
HOME_URL="https://amazonlinux.com/"
```



