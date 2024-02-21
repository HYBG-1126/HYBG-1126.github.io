---
categories: [Kubernetes]
tags: Kubernetes
---


# kubernetes常用命令

## 创建资源类型

<details>

<summary>创建一般quota</summary>

#### 编写quota.yaml
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
spec:
  hard:
    requests.cpu: "200"
    requests.memory: 600Gi
    limits.cpu: "200"
    limits.memory: 600Gi
```

#### 执行命令创建quota\
```shell
kubectl apply -f quota.yaml -n namespace
```

</details>




### 创建不同优先级的quota资源，[Pod 优先级和抢占](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/pod-priority-preemption/)
<details>
  <summary>编写PriorityClass.yaml</summary>

``` yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
  namespace: bigdata
value: 1000000
globalDefault: false
description: "High priority class for bigdata-pro namespace pods"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium
  namespace: bigdata
value: 100000
globalDefault: false
description: "Medium priority class for bigdata-pro namespace pods"
```

</details>

<details>
<summary>执行命令创建PriorityClass</summary>

```shell
kubectl apply -f PriorityClass.yaml
```

</details>


<details>
    <summary>编写各个优先级的资源大小resourequota.yaml</summary>

``` yaml
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: bigdata-high
      namespace: bigdata
    spec:
      hard:
        limits.cpu: "20"
        limits.memory: 50Gi
        requests.cpu: "20"
        requests.memory: 50Gi
      scopeSelector:
        matchExpressions:
          - operator: In
            scopeName: PriorityClass
            values: ["high"]
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: bigdata-medium
      namespace: bigdata
    spec:
      hard:
        limits.cpu: "100"
        limits.memory: 200Gi
        requests.cpu: "100"
        requests.memory: 200Gi
      scopeSelector:
        matchExpressions:
          - operator: In
            scopeName: PriorityClass
            values: ["medium"]
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: namespace-quota
      namespace: bigdata
    spec:
      hard:
        requests.cpu: "120"
        requests.memory: 250Gi
        limits.cpu: "120"
        limits.memory: 250Gi
```
</details>
<details>
 <summary>执行命令创建resourequota</summary>

```shell
kubectl apply -f resourequota.yaml
```
</details>



## 查看资源使用情况
### 查看节点资源使用情况
```shell
# 查看使用情况
kubectl top node

# 查看使用情况并排序
kubectl top node --sort-by='memory'/'cpu'
```
### 查看pod资源使用情况
```shell
# 查看使用情况
kubectl top pod

# 查看使用情况并排序
kubectl top pod --sort-by='memory'/'cpu'
```



 

 
