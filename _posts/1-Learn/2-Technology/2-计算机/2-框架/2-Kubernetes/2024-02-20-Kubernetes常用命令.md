---
categories: [Kubernetes]
tags: Kubernetes
---


# kubernetes常用命令

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



 

 
