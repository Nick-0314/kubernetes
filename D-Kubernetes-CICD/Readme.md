# Kubernetes-CICD

本文介绍了kubernetes的持续集成持续交付持续部署

本文分A B两篇，A篇采用二进制Kubernetes1.16+harbor独立部署+gitlab独立部署+jenkins独立部署

A篇环境：

| 主机角色          | IP地址        |
| ----------------- | ------------- |
| kubernetes-master | 192.168.10.10 |
| kubernetes-node1  | 192.168.10.11 |
| kubernetes-node2  | 192.168.10.12 |
| gitlab            | 192.168.10.3  |
| jenkins           | 192.168.10.2  |
| Harbor            | 192.168.10.66 |

