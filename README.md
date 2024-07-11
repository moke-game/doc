# 从0-1设计云原生游戏微服务架构：

本系列文章主要描述如何实战架构一个高可用的云原生游戏微服务架构      
采用golang作为主要开发语言，grpc(同步)/nats(异步)作为服务间通信机制，   
k8s作为容器化部署工具，以及各种云原生工具来实现自动化部署和监控。

教程源码可以参见:[moke-game](https://github.com/orgs/moke-game/repositories)

## 应用场景:

主要适用各种游戏类型的架构，包括MMO，MOBA，FPS等房间制的游戏

## 包含的内容：

* 规范开发流程
* 服务的拆分
* 战斗/大世界服务实现
* 服务安全：服务间的mTLS认证、基于token的用户权限认证
* 服务间的通信: grpc(同步)、nats(异步)
* 数据库的一致性保证：基于version的CAS乐观锁
* 容器化+CI/CD自动化
* k8s集群管理和监控
* 服务的压力测试

## 相关技术:

* 基础服务框架(自己写的^_^,欢迎star)：[moke-kit](https://github.com/GStones/moke-kit)
* [grpc](https://grpc.io/),[nats](https://nats.io/): 实现服务间的同步/异步通信
* [agones](https://agones.dev/site/),[openmatch](https://open-match.dev/site/): 实现战斗房间的托管以及分配
* docker,harbor,jenkins,argoCD: 实现容器化以及CI/CD自动化
* k8s, helm ,k9s: k8s集群管理