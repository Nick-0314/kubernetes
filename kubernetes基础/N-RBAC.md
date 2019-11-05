# RBAC

## 1 RBAC基本概念

RBAC(Role-Based Access Control,基于角色的访问控制)是一种基于企业内个人用户角色来管理对计算机或网络资源的访问方法，其在Kuberentes1.5版本种引入，在1.6时升级为Beta版版，并成为Kubeadm安装方式下的默认选项。启用RBAC需要在启动APIServer时指定--authorization-mode=RBAC.

RBAC使用rbac.authorization.K8S.io API组来推动授权决策，允许管理员通过Kubernetes API动态配置策略。

RBAC API声明了4种顶级资源对象，即Role，ClusterRole，RoleBInding，管理员可以像使用其他API资源一样使用kubectl API调用这些资源对象。例如：kubectl create -f (resource).yaml。

## 2 Role和ClusterRole

Role和ClusterRole的关键区别是，Role是作用于命名空间内的角色，ClusterRole作用于整个集群的角色。

在RBAC API中，Role包含表示一组权限的规则。权限纯粹是附加允许的，没有拒绝规则。

Role只能授权对单个命名空间内的资源的访问权限，比如授权对default命名空间的读取权限：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
```

ClusterRole也可将上诉权限授予作用于整个集群的Role，主要区别是，ClusterRole是集群范围的，因此它们还可以授予对以下内容的访问权限：

- 集群范围的资源(如Node)。
- 非资源端点（如/healthz）.
- 跨所有命名空间的命名空间资源(如Pod)。

比如，授予对任何特定命名空间或所有命名空间中的secret的读权限(取决于它的绑定形式):

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get","watch","list"]
```

可以通过如下命令查看创建的RBAC

```
kubectl get role.rbac.authorization.k8s.io
```

输出信息

```
NAME            AGE
pod-reader      6m14s
secret-reader   20s
```

## 3 RoleBinding和ClusterRoleBinding

RoleBinding将Role中定义的权限授予User，Group或Service Account。RoleBinding和ClusterRoleBinding最大的区别与Role和ClusterRole的区别类似，即RoleBinding作用于命名空间，ClusterRoleBinding作用于集群。

RoleBinding可以引用同一命名空间的Role进行授权，比如将上述创建的pod-reader的Role授予default命名空间的用户jane,这将允许jane读取default命名空间中的Pod：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

说明：

- roleRef: 绑定的类别，可以是Role或ClusterRolei

RoleBinding也可以引用ClusterRole来授予对命名空间资源的某些权限。管理员可以为整个集群定义一组公用的ClusterRole，然后在多个命名空间中重复使用。

 比如，创建一个RoleBinding引用ClusterRole，授予dave用户读取development命名空间的Secret:

先创建一个namespace：

```
kubectl create namespace development
```

输出信息：

```
namespace/development created
```

yaml文件

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

ClusterRoleBinding可用于在集群及级别和所有命名空间中授予权限，比如允许组manager中的所有用户都能读取任何命名空间的Secret：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## 4 对集群资源的权限控制

在Kubernetes中，大多数资源都由其名称的字符串表示，例如pods。但是一些Kubernetes API 涉及的子资源(下级资源)，例如Pod的日志，对应的Endpoint的URL是：

```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

在这种情况下，pods是命名空间资源，log是Pod的下级资源，如果对其进行访问控制，要使用斜杠来分隔资源和子资源，比如定义一个Role允许读取Pod和Pod日志：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-and-pod-logs-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get","list"]
```

针对具体资源（使用  resourceNames指定单个具体资源）的某些请求，也可以通过使用get，delete，update，patch等进行授权，比如，只能对一个叫my-configmap的configmap进行get和update操作：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-updater
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["get","update"]
```

如果使用了resourceNames，则verbs不能是list,watch,create,delete,conllection等。

## 5 聚合ClusterRole

从Kubernetes1.9版本开始，Kubernetes可以通过一组ClusterRole创建聚合ClusterRoles，聚合ClusterRoles的权限由控制器管理，并通过匹配ClusterRole的标签自动填充相对应的权限。

比如，匹配rbac.example.com/aggregate-to-monitoring:"true"标签来创建聚合ClusterRole

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []
```

然后创建与标签选择器相匹配的ClusterRole向聚合ClusterRole添加规则:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["service","pods","endpoints"]
  verbs: ["get","list","watch"]
```

## 6 Role示例

以下示例允许读取核心API组中的资源Pods(只写了规则rules部分)：

```
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
```

允许在extensions和apps API组中读写deployments：

```
rules:
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get","list","watch","create","update","patch","delete"]
```

允许对Pods的读和Job的读写：

```
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
- apiGroups: ["batch","extensions"]
  resources: ["jobs"]
  verbs: ["get","list","watch","create","update","patch","delete"]
```

允许读取一个名为my-config的ConfigMap(必须绑定到一个RoleBindiong来限制到一个命名空间下的ConfigMap)：

```
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

允许读取核心组Node资源(Node属于集群级别的资源，必须放在ClusterRole中，并使用ClusterRoleBinding进行绑定)：

```
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get","list","watch"]
```

允许对非资源端点/healthz和所有其子资源的路径的Get和Post请求（必须放在ClusterRole并与ClusterRoleBinding进行绑定）：

```
rules:
- nonResourceURLs: ["/healthz","/healthz/*"]
  verbs: ["get","post"]
```

## 7 RoleBinding示例

以下示例绑定名为“alice@exampli.com”的用户(只显示subjexts部分):

```
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
```

绑定名为“frontend-admins”的组：

```
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
```

绑定为kube-system命名空间中的默认Service Account:

```
subjects:
- kind: ServiceAccout
  name: default
  namespace: kube-system
```

绑定为qa命名空间中的所有Service Account:

```
subjects:
- kind: Group
  name: system:serviceaccouts:qa
  apiGroup: rbac.authorization.k8s.io
```

绑定所有Service Account：

```
subjects:
- kind: Group
  name: system:serviceaccouts
  apiGroup: rbac.authorization.k8s.io
```

绑定所有经过身份验证的用户(v1.5+)

```
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

绑定所有未经过身份验证的用户(v1.5+)

```
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

绑定所有用户：

```
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

## 8 命令行的使用

权限的创建可以使用命令行直接创建 ，较上述方式更加简单，快捷，

1 kubectl create role

创建一个Role，命名为pod-reader，允许用户在Pod上执行getl,watch和list