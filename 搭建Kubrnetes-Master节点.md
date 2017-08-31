# 部署Kubernetes Master节点

## kube-apiserver组件
kube-apiserver是Kubernetes最重要的核心组件之一，主要提供以下的功能：

+ 提供集群管理的REST API接口，包括认证授权、数据校验以及集群状态变更等
+ 提供其他模块之间的数据交互和通信的枢纽(其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd)。

## kube-scheduler组件
kube-scheduler是Kubernetes最重要的核心组件之一，主要提供以下的功能:
+ 负责分配调度Pod到集群内的节点上，它监听kube-apiserver，查询还未分配Node的Pod，然后根据调度策略为这些Pod分配节点(更新Pod的NodeName字段)。

## kube-controller-manager组件
Controller Manager是Kubernetes最重要的核心组件之一，主要提供以下的功能：
+ 主要kube-controller-manager和cloud-controller-manager组成，是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。
