---
layout: post
title: 阿里云日志服务采集k8s日志并实现livetail功能
date: 2018-11-15
tags:
  - aliyun 
  - 日志组件
  - livetail
  - k8s
lang: zh-Hans
author: keking.com
---

## 前言
目前的项目日志都是通过Logtail直接采集，投递到OSS持久化，同时可以通过阿里云日志服务、devops自建平台进行查看（虽然大部分人是直接登录ECS查看=。=）
在开始进行容器化之后，同样遇到日志的问题，目前的解决方案是阿里云日志服务持久化和展现格式化后的日志、使用rancher查看实时日志
但是之前由于rancher平台出现一些问题，导致不能及时查看日志的情况，在这个背景下对阿里云日志服务采集k8s日志和livetail进行搭建并调研词方案是否可行


## 简介（转自阿里云官方文档）
> 日志服务（Log Service，简称 LOG）是针对日志类数据的一站式服务，在阿里巴巴集团经历大量大数据场景锤炼而成。您无需开发就能快捷完成日志数据采集、消费、投递以及查询分析等功能，提升运维、运营效率，建立 DT 时代海量日志处理能力。

## kubernetes日志采集组件安装
### 安装Logtail
  + 进入阿里云容器服务找到集群id
  <figure>
  <a><img src="{{site.url}}/uploads/20181115/001.jpeg"></a>
  </figure>
  
  + 通过ssh登录master节点，或者任意安装了kubectl并配置了该集群kubeconfig的服务器
  + 执行命令，将${your_k8s_cluster_id}替换为集群id

  ```
   wget http://logtail-release-cn-hangzhou.oss-cn-hangzhou.aliyuncs.com/kubernetes/alicloud-log-k8s-install.sh -O alicloud-log-k8s-install.sh; chmod 744 ./alicloud-log-k8s-install.sh; sh ./alicloud-log-k8s-install.sh ${your_k8s_cluster_id}
  ```
    
    Project k8s-log-${your_k8s_cluster_id}下会自动创建名为config-operation-log的Logstore，用于存储alibaba-log-controller的运行日志。
    请勿删除此Logstore，否则无法为alibaba-log-controller排查问题。
    
    若您需要将日志采集到已有的Project，请执行安装命令sh ./alicloud-log-k8s-install.sh${your_k8s_cluster_id} ${your_project_name} ，并确保日志服务Project和您的Kubernetes集群在同一地域。
 
  - 该条命令其实就是执行了一个shell脚本，使用helm安装了采集kubernetes集群日志的组件

  ```
      #!/bin/bash
     
    if [ $# -eq 0 ] ; then
        echo "[Invalid Param], use sudo ./install-k8s-log.sh {your-k8s-cluster-id}"
        exit 1
    fi
     
    clusterName=$(echo $1 | tr '[A-Z]' '[a-z]')
    curl --connect-timeout 5  http://100.100.100.200/latest/meta-data/region-id
     
    if [ $? != 0 ]; then
        echo "[FAIL] ECS meta server connect fail, only support alibaba cloud k8s service"
        exit 1
    fi
     
    regionId=`curl http://100.100.100.200/latest/meta-data/region-id`
    aliuid=`curl http://100.100.100.200/latest/meta-data/owner-account-id`
     
    helmPackageUrl="http://logtail-release-$regionId.oss-$regionId.aliyuncs.com/kubernetes/alibaba-cloud-log.tgz"
    wget $helmPackageUrl -O alibaba-cloud-log.tgz
    if [ $? != 0 ]; then
        echo "[FAIL] download alibaba-cloud-log.tgz from $helmPackageUrl failed"
        exit 1
    fi
     
    project="k8s-log-"$clusterName
    if [ $# -ge 2 ]; then
        project=$2
    fi
     
    echo [INFO] your k8s is using project : $project
     
    helm install alibaba-cloud-log.tgz --name alibaba-log-controller \
        --set ProjectName=$project \
        --set RegionId=$regionId \
        --set InstallParam=$regionId \
        --set MachineGroupId="k8s-group-"$clusterName \
        --set Endpoint=$regionId"-intranet.log.aliyuncs.com" \
        --set AlibabaCloudUserId=":"$aliuid \
        --set LogtailImage.Repository="registry.$regionId.aliyuncs.com/log-service/logtail" \
        --set ControllerImage.Repository="registry.$regionId.aliyuncs.com/log-service/alibabacloud-log-controller"
     
    installRst=$?
     
    if [ $installRst -eq 0 ]; then
        echo "[SUCCESS] install helm package : alibaba-log-controller success."
        exit 0
    else
        echo "[FAIL] install helm package failed, errno " $installRst
        exit 0
    fi
  ```
  - 命令执行后，会在kubernetes集群中的每个节点运行一个日志采集的pod：logatail-ds，该pod位于kube-system
  <figure>
  <a><img src="{{site.url}}/uploads/20181115/002.jpeg"></a>
  </figure>
  - 安装完成后，可使用以下命令来查看pod状态，若状态全部成功后，则表示安装完成

  ```
    [root@izuf6d75c59ll4woscqmh5z ~]# helm status alibaba-log-controller
    LAST DEPLOYED: Thu Nov 22 15:09:35 2018
    NAMESPACE: default
    STATUS: DEPLOYED
     
    RESOURCES:
    ==> v1/ServiceAccount
    NAME                    SECRETS  AGE
    alibaba-log-controller  1        6d
     
    ==> v1beta1/CustomResourceDefinition
    NAME                                   AGE
    aliyunlogconfigs.log.alibabacloud.com  6d
     
    ==> v1beta1/ClusterRole
    alibaba-log-controller  6d
     
    ==> v1beta1/ClusterRoleBinding
    NAME                    AGE
    alibaba-log-controller  6d
     
    ==> v1beta1/DaemonSet
    NAME        DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
    logtail-ds  16       16       16     16          16         <none>         6d
     
    ==> v1beta1/Deployment
    NAME                    DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    alibaba-log-controller  1        1        1           1          6d
     
    ==> v1/Pod(related)
    NAME                                     READY  STATUS   RESTARTS  AGE
    logtail-ds-2fqs4                         1/1    Running  0         6d
    logtail-ds-4bz7w                         1/1    Running  1         6d
    logtail-ds-6vg88                         1/1    Running  0         6d
    logtail-ds-7tp6v                         1/1    Running  0         6d
    logtail-ds-9575c                         1/1    Running  0         6d
    logtail-ds-bgq84                         1/1    Running  0         6d
    logtail-ds-kdlhr                         1/1    Running  0         6d
    logtail-ds-lknxw                         1/1    Running  0         6d
    logtail-ds-pdxfk                         1/1    Running  0         6d
    logtail-ds-pf4dz                         1/1    Running  0         6d
    logtail-ds-rzsnw                         1/1    Running  0         6d
    logtail-ds-sqhbv                         1/1    Running  0         6d
    logtail-ds-vvtwn                         1/1    Running  0         6d
    logtail-ds-wwmhg                         1/1    Running  0         6d
    logtail-ds-xbp4j                         1/1    Running  0         6d
    logtail-ds-zpld9                         1/1    Running  0         6d
    alibaba-log-controller-85f8fbb498-nzhc8  1/1    Running  0         6d

    ```

    ### 配置日志组件展示
    - 在集群内安装好日志组件后，登录阿里云日志服务控制台，就会发现有一个新的project，名称为k8s-log-{集群id}
      <figure>
  <a><img src="{{site.url}}/uploads/20181115/003.png"></a>
  </figure>
  
  - 创建Logstore
       <figure>
  <a><img src="{{site.url}}/uploads/20181115/004.png"></a>
  </figure>

  - 配置Logstore，设置logstore名称、配置日志数据保留时间
       <figure>
  <a><img src="{{site.url}}/uploads/20181115/005.png"></a>
  </figure>

  - 数据导入
      <figure>
  <a><img src="{{site.url}}/uploads/20181115/006.png"></a>
  </figure>
 
  - 选择数据类型中选择docker标准输出
  <figure>
  <a><img src="{{site.url}}/uploads/20181115/007.png"></a>
  </figure>

  - 数据源配置，这里可以使用默认的
  <figure>
  <a><img src="{{site.url}}/uploads/20181115/008.png"></a>
  </figure>

  - 选择数据源
  <figure>
  <a><img src="{{site.url}}/uploads/20181115/009.png"></a>
  </figure>

  - 配置好之后等待1-2分钟，日志就会进来了
 <figure>
  <a><img src="{{site.url}}/uploads/20181115/010.png"></a>
  </figure>

  - 为了快速查询和过滤，需要配置索引

 <figure>
  <a><img src="{{site.url}}/uploads/20181115/011.png"></a>
  </figure>

  - 添加容器名称、命名空间、pod名称作为索引（后续使用livetail需要）
 <figure>
  <a><img src="{{site.url}}/uploads/20181115/012.png"></a>
  </figure>
  - 这样就完成了一个k8s集群日志采集和展示的基本流程了

  ## livetail功能使用

  ### 背景介绍
    > 在线上运维的场景中，往往需要对日志队列中进入的数据进行实时监控，从最新的日志数据中提取出关键的信息进而快速地分析出异常原因。在传统的运维方式中，如果需要对日志文件进行实时监控，需要到服务器上对日志文件执行命令tail -f，如果实时监控的日志信息不够直观，可以加上grep或者grep -v进行关键词过滤。日志服务在控制台提供了日志数据实时监控的交互功能LiveTail，针对线上日志进行实时监控分析，减轻运维压力。

  ### 使用方法
  - 这里选择来源类型为kubernetes，命名空间、pod名称、容器名称为上一步新建的3个索引的内容，过滤关键字的功劳与tail命令后加的grep命令是一样的，用于关键词过滤

   <figure>
  <a><img src="{{site.url}}/uploads/20181115/013.png"></a>
  </figure>

  - 点击开启livetail，这时就有实时日志展示出来了
     <figure>
  <a><img src="{{site.url}}/uploads/20181115/014.png"></a>
  </figure>

## 问题与短板
阿里云日志服务的livetail功能比起传统的ELK日志展示强大了许多，但是还是存在具体操作需要手动查找、拷贝命名空间、pod名称、容器名称，虽然命名空间和容器名称为固定的，但是pod名称需要查找当前pod的名称，操作起来比较繁琐，功能方面确实不如rancher的实时日志展示，并提供进入终端查看日志文件的功能强大。

同时该功能才测试使用的时候还出现过网络连接错误，闪退等情况，总体来说属于阿里云日志服务的一项新功能，不适合直接给开发运维来使用。


## 总结
由于公司一直使用阿里云的日志服务，总体来说，kubernetes日志采集和展示服务的接入和配置速度很快，如果有需要可以随时拉起并投入使用。

对于现阶段有更方便使用的容器化展示平台的情况下，阿里云livetail方案可以作为一个备选方案，一旦rancher出现问题，可以所以接入和配置使用。

总的来说此方案不如rancher使用灵活和功能强大，不建议推广使用，可作为一个备选方




