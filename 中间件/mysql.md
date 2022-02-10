## 基础

## 集群架构

- 主从  
  默认从只做备份，但是可以通过配置达到读写分离的效果


- 双主 2个互为主从  
  ![](../resources/mysql.jpg)

- MHA   
  总架构  
  ![](../resources/mysql1.jpg)

  组内架构  
  ![](../resources/mysql2.jpg)

  MHA Manager可以单独部署 在一台独立的机器上管理多个master-slave集群，也可以部署在一台slave节点上。MHA Node运行在每台 MySQL服务器上，MHA
  Manager会定时探测集群中的master节点，当master出现故障时，它可以自动将最新 数据的slave提升为新的master，然后将所有其他的slave重新指向新的master。整个故障转移过程对应用程序完 全透明。

  假设每个mysql组配置如下一主二从：
  ![](../resources/mysql3.jpg)

  其中master对外提供写服务，备选master（实际的slave，主机名slave1）提供读服务，slave也提供相关的读服务，一旦master宕机，将会把备选master提升为新的master，slave指向新的master，manager作为管理服务器。