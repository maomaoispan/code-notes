# 1 K8s概念和架构

#### 功能：

- 自动装箱
- 自我修复
- 水平扩展
- 服务发现
- 滚动更新
- 版本回退
- 配置
- 存储处理



# 2 从零搭建K8s集群

### Master（主控）节点：

#### API server

​	restful -》etcd

#### controller-manager：

​	集中控制管理

#### scheduler：

​	集群调度

#### etcd

存储系统，保存集群相关数据

### Node（工作）节点

#### kubelet

​	节点代表，管理本机容器

#### kube-proxy

​	网络代理



### 平台规划

硬件要求：

master： CUP 2核心 内存 4G 20G

node:  4核心 8G 40G

## 1.1 基于客户端工具 kubeadm

### 节点配置

k8s-node-1：8.142.37.99（master）

k8s-node-2：39.103.233.210

k8s-node-3：39.99.147.1



## 1.2 基于二进制包



# 3 k8s 核心概念

#### Pod

- 共享网络、
- 最小单元、
- 容器的集合，
- 生命周期短暂

#### Controller

- 确保pod数量
- 有状态应用部署
- 无状态应用部署
- 确保所有node运行同一个pod
- 一次性、定时任务

#### Service

- 定义pod访问规则

#### Ingress

#### RABC 

#### Helm

#### 持久存储器



# 4 搭建集群监控平台系统



# 5 从零开始搭建高可用k8s集群



# 6 在集群环境部署项目

