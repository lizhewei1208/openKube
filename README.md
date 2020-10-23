# openKube
自定义CRD资源demo，kubebuilder v2.3.1

## 1、初始化项目目录

```go
export GO111MODULE=on   // 开启gomod

mkdir openkube    // 在gopath下面创建项目目录

kubebuilder init --domain openkube.com   // domain指定CRD资源group的域名,

注意：domain需要全部小写
```

## 2、API创建

```go
kubebuilder create api --group apps --version v1beta1 --kind UnitedSet --namespaced true 
// 实际上不仅会创建 API，也就是 CRD，还会生成 Controller 的框架

Create Resource [y/n]
y
Create Controller [y/n]
y
```

**参数解读**：- group 加上之前的 domian 即此 CRD 的 Group: apps.kruise.io；

- version 一般分三种，按社区标准：
  - v1alpha1: 此 api 不稳定，CRD 可能废弃、字段可能随时调整，不要依赖；
  - v1beta1: api 已稳定，会保证向后兼容，特性可能会调整；
  - v1: api 和特性都已稳定；
- kind: 此 CRD 的类型，类似于社区原生的 Service 的概念；
- namespaced: 此 CRD 是k8s集群级别资源还是 namespace隔离的，类似 node 和 Pod

## 3、创建webhook（可选）

https://www.qikqiak.com/post/k8s-admission-webhook/

```go
// 创建pod的 MutatingAdmissionWebhook
kubebuilder create webhook --group core --version v1 --kind Pod --defaulting

// 创建UnitedSet的 MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook
kubebuilder create webhook --group apps --version v1beta1 --kind UnitedSet --defaulting --programmatic-validation
```
## 4、定义 CRD

在图 1中对应的文件定义 Spec 和 Status。

## 5、编写 Controller 和wenbook逻辑

在图 1 中对应的文件实现 Reconcile 以及webhook逻辑。

## 6、本地调试运行
### 6.1、CRD安装

```go
开启go mod模式
export GO111MODULE=on
export GOPROXY="http://mirrors.aliyun.com/goproxy"
export GOPROXY="https://goproxy.cn"

第一种方式：
将代码拷贝至k8s集群机器
make install    // 先生成config/crd/bases文件，里面包含crd的YAML文件，然后kubectl apply

第二种方式: 
在本地机器执行，make install，在config/crd/bases目录生成crd.yaml，拷贝至k8s集群机器，执行

kubectl create -f apps.openkube.com_unitedsets.yaml
```

然后我们就可以看到创建的CRD了

```
# kubectl get crd
NAME                           CREATED AT
unitedsets.apps.openkube.com   2020-10-15T03:35:45Z
```

创建一个unitedSet资源

```
# cd openKube/config/samples

# kubectl create -f apps_v1beta1_unitedset.yaml

# kubectl get unitedsets.apps.openkube.com
NAME               AGE
unitedset-sample   67s
```

看一眼yaml文件

```
[root@k8s-master samples]# cat apps_v1beta1_unitedset.yaml 
apiVersion: apps.openkube.com/v1beta1
kind: UnitedSet
metadata:
  name: unitedset-sample
spec:
  # Add fields here
  foo: bar
```

这里仅仅是把yaml存到etcd里了，我们controller监听到创建事件时啥事也没干。
### 6.2、修改配置文件

因为启用了webhook，所以要对默认的配置文件进行一些修改，来到config目录，config的核心是config/default目录。

#### 6.2.1、修改config/default/kustomization.yaml
```
# Adds namespace to all resources.
namespace: openkube-system

# Value of this field is prepended to the
# names of all resources, e.g. a deployment named
# "wordpress" becomes "alices-wordpress".
# Note that it should also match with the prefix (text before '-') of the namespace
# field above.
namePrefix: openkube-

# Labels to add to all resources and selectors.
#commonLabels:
#  someName: someValue

bases:
- ../crd
- ../rbac
- ../manager
# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in 
# crd/kustomization.yaml
- ../webhook
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'. 'WEBHOOK' components are required.
#- ../certmanager
# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'. 
#- ../prometheus

patchesStrategicMerge:
  # Protect the /metrics endpoint by putting it behind auth.
  # If you want your controller-manager to expose the /metrics
  # endpoint w/o any authn/z, please comment the following line.
- manager_auth_proxy_patch.yaml

# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in 
# crd/kustomization.yaml
#- manager_webhook_patch.yaml

# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'.
# Uncomment 'CERTMANAGER' sections in crd/kustomization.yaml to enable the CA injection in the admission webhooks.
# 'CERTMANAGER' needs to be enabled to use ca injection
#- webhookcainjection_patch.yaml

# the following config is for teaching kustomize how to do var substitution
vars:
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER' prefix.
#- name: CERTIFICATE_NAMESPACE # namespace of the certificate CR
#  objref:
#    kind: Certificate
#    group: cert-manager.io
#    version: v1alpha2
#    name: serving-cert # this name should match the one in certificate.yaml
#  fieldref:
#    fieldpath: metadata.namespace
#- name: CERTIFICATE_NAME
#  objref:
#    kind: Certificate
#    group: cert-manager.io
#    version: v1alpha2
#    name: serving-cert # this name should match the one in certificate.yaml
- name: SERVICE_NAMESPACE # namespace of the service
  objref:
    kind: Service
    version: v1
    name: webhook-service
  fieldref:
    fieldpath: metadata.namespace
- name: SERVICE_NAME
  objref:
    kind: Service
    version: v1
    name: webhook-service
```

#### 6.2.2、修改config/crd/kustomization.yaml文件
```
# This kustomization.yaml is not intended to be run by itself,
# since it depends on service name and namespace that are out of this kustomize package.
# It should be run by config/default
resources:
- bases/apps.openkube.com_unitedsets.yaml
# +kubebuilder:scaffold:crdkustomizeresource

patchesStrategicMerge:
# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix.
# patches here are for enabling the conversion webhook for each CRD
- patches/webhook_in_unitedsets.yaml
# +kubebuilder:scaffold:crdkustomizewebhookpatch

# [CERTMANAGER] To enable webhook, uncomment all the sections with [CERTMANAGER] prefix.
# patches here are for enabling the CA injection for each CRD
#- patches/cainjection_in_unitedsets.yaml
# +kubebuilder:scaffold:crdkustomizecainjectionpatch

# the following config is for teaching kustomize how to do kustomization for CRDs.
configurations:
- kustomizeconfig.yaml
```
### 7、修改Makefile
```
# Deploy controller in the configured Kubernetes cluster in ~/.kube/config
deploy: manifests
        cd config/manager && kustomize edit set image controller=${IMG}
#       kustomize build config/default | kubectl apply -f -
        kustomize build config/default > all_in_one.yaml
```
可以看到，此命令会使用kustomize订制整个config/default目录下的配置文件，生成所有的资源文件，再使用kubectl apply命令部署，但直接apply在部分版本的K8s中可能会出错。为了更清晰地了解kustomize生成的资源有哪些，我将它做了一些小修改，不直接apply，转而将资源重定向到all_in_one.yaml文件内。
### all_in_one
#### 分析
仔细分析一番生成的all_in_one.yaml文件，有6000多行，其中的CustomResourceDefinition资源占据绝大部分的内容，总共可大概有这几种类型的资源:
```
# CRD的资源描述,涉及到Unit的每一个字段，因此非常冗长.
kind: CustomResourceDefinition

# admission webhook
kind: MutatingWebhookConfiguration
kind: ValidatingWebhookConfiguration

# RBAC授权
kind: Role
kind: ClusterRole
kind: RoleBinding
kind: ClusterRoleBinding

# prometheus metric service
kind: Service

# openkube-webhook-service，接收APIServer的回调
kind: Service

# openkube controller deployment
kind: Deployment
```
#### 修改yaml文件
##### 需要把yaml文件中CustomResourceDefinition.spec下新增一个字段：`preserveUnknownFields: false`
否则不加此字段kubectl apply会报错，bug已知存在于1.15-1.17以下的版本中，参考: Generated Metadata breaks crd
```
apiVersion: v1
kind: Namespace
metadata:
  labels:
    control-plane: controller-manager
  name: openkube-system
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.2.5
  name: unitedsets.apps.openkube.com
spec:
  preserveUnknownFields: false                           // 增加此参数
  conversion:
    strategy: Webhook
    webhookClientConfig:
...
```
##### 修改MutatingWebhookConfiguration 和 ValidatingWebhookConfiguration
这两个webhook配置需要修改什么呢？来看看下载的配置，以为例：MutatingWebhookConfiguration
```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  creationTimestamp: null
  name: openkube-mutating-webhook-configuration
webhooks:
- clientConfig:
    caBundle: Cg=
    service:
      name: openkube-webhook-service
      namespace: openkube-system
      path: /mutate-pod
  failurePolicy: Fail
  name: mpod.kb.io
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods
```
这里面有两个地方要修改：

- caBundle现在是空的，需要补上
- clientConfig现在的配置是ca授权给的是Service unit-webhook-service，也即是会转发到deployment的pod，但我们现在是要本地调试，这里就要改成本地环境。

下面来讲述如何配置这两个点。
###### CA证书签发
这里要分为多个步骤：

**1.ca.cert**

首先获取K8s CA的CA.cert文件：

```
kubectl config view --raw -o json | jq -r '.clusters[0].cluster."certificate-authority-data"' | tr -d '"' > ca.cert

```

ca.cert的内容，即可复制替换到上面的MutatingWebhookConfiguration和ValidatingWebhookConfigurationd的`webhooks.clientConfig.caBundle`里。(原来的`Cg==`要删掉.)

####### 2.csr

创建证书签署请求json配置文件：

注意，hosts里面填写两种内容：

- openkube controller的service 在K8s中的域名，最后openkube controller是要放在K8s里运行的。
- 本地开发机的某个网卡IP地址，这个地址用来连接K8s集群进行调试。因此必须保证这个IP与K8s集群可以互通

```shell
cat > openkube-csr.json << EOF
{
  "hosts": [
    "openkube-webhook-service.default.svc",
    "openkube-webhook-service.default.svc.cluster.local",
    "192.168.254.1"
  ],
  "CN": "openkube-webhook-service",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF
```

####### 3.生成csr和pem私钥文件:

```shell
[root@k8s-master deploy]# cat openkube-csr.json | cfssl genkey - | cfssljson -bare openkube
2020/05/23 17:44:39 [INFO] generate received request
2020/05/23 17:44:39 [INFO] received CSR
2020/05/23 17:44:39 [INFO] generating key: rsa-2048
2020/05/23 17:44:39 [INFO] encoded CSR
[root@k8s-master deploy]# ls
openkube.csr  openkube-csr.json  openkube-key.pem
```

####### 4.创建CertificateSigningRequest资源

```shell
cat > csr.yaml << EOF 
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: openkube
spec:
  request: $(cat openkube.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF

# apply
kubectl apply -f csr.yaml
```

####### 5.向集群提交此CertificateSigningRequest.

查看状态：

```shell
[root@k8s-master deploy]# kubectl apply -f csr.yaml 
certificatesigningrequest.certificates.k8s.io/openkube created
[root@k8s-master deploy]# kubectl describe csr openkube 
Name:         openkube
Labels:       <none>
...

CreationTimestamp:  Tue, 20 Oct 2020 23:37:47 -0400
Requesting User:    kubernetes-admin
Signer:             kubernetes.io/legacy-unknown
Status:             Pending
Subject:
  Common Name:    openkube-webhook-service
  Serial Number:  
Subject Alternative Names:
         DNS Names:     openkube-webhook-service.default.svc
                        openkube-webhook-service.default.svc.cluster.local
         IP Addresses:  10.200.224.94
Events:  <none>
```

可以看到它还是pending的状态，需要同意一下请求:

```shell
[root@k8s-master deploy]#  kubectl certificate approve openkube
certificatesigningrequest.certificates.k8s.io/openkube approved
[root@k8s-master deploy]# kubectl get csr openkube 
NAME       AGE     SIGNERNAME                     REQUESTOR          CONDITION
openkube   2m15s   kubernetes.io/legacy-unknown   kubernetes-admin   Approved,Issued
# 保存客户端crt文件
[root@k8s-master deploy]# kubectl get csr openkube -o jsonpath='{.status.certificate}' | base64 --decode > openkube.crt
```

可以看到，现在已经签署完毕了。

汇总一下：

- 第1步生成的ca.cert文件给caBundle字段使用
- 第3步生成的unit-key.pem私钥文件和第5步生成的unit.crt文件，提供给客户端(unit controller)https服务使用

