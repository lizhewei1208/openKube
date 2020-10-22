# openKube
自定义CRD资源demo，kubebuilder v2.3.1

#### 1、初始化项目目录

```go
export GO111MODULE=on   // 开启gomod

mkdir openkube    // 在gopath下面创建项目目录

kubebuilder init --domain openkube.com   // domain指定CRD资源group的域名,

注意：domain需要全部小写
```

#### 2、API创建

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

#### 3、创建webhook（可选）

https://www.qikqiak.com/post/k8s-admission-webhook/

```go
// 创建pod的 MutatingAdmissionWebhook
kubebuilder create webhook --group core --version v1 --kind Pod --defaulting

// 创建UnitedSet的 MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook
kubebuilder create webhook --group apps --version v1beta1 --kind UnitedSet --defaulting --programmatic-validation
```
#### 4、定义 CRD

在图 1中对应的文件定义 Spec 和 Status。

#### 5、编写 Controller 和wenbook逻辑

在图 1 中对应的文件实现 Reconcile 以及webhook逻辑。

#### 6、本地调试运行
##### 6.1、CRD安装

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
##### 6.2、修改配置文件

因为启用了webhook，所以要对默认的配置文件进行一些修改，来到config目录，config的核心是config/default目录。

###### 6.2.1、修改config/default/kustomization.yaml
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

###### 6.2.2、修改config/crd/kustomization.yaml文件
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
##### 7、修改Makefile
```
# Deploy controller in the configured Kubernetes cluster in ~/.kube/config
deploy: manifests
        cd config/manager && kustomize edit set image controller=${IMG}
#       kustomize build config/default | kubectl apply -f -
        kustomize build config/default > all_in_one.yaml
```
可以看到，此命令会使用kustomize订制整个config/default目录下的配置文件，生成所有的资源文件，再使用kubectl apply命令部署，但直接apply在部分版本的K8s中可能会出错。为了更清晰地了解kustomize生成的资源有哪些，我将它做了一些小修改，不直接apply，转而将资源重定向到all_in_one.yaml文件内。



