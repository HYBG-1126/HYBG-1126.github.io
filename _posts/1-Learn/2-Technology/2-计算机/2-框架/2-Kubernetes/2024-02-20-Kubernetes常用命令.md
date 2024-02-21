---
categories: [Kubernetes]
tags: Kubernetes
---


# kubernetes常用命令

## 创建资源类型
### 创建quota
#### 编写quota.yaml
```shell
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
#### 创建quota
```shell
kubectl apply -f quota.yaml -n namespace
```

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



 

 
