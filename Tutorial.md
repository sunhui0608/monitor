# 监控搭建文档

本文主要概述CCS搭建监控运用的关键技术、原理及步骤。

## 1. 技术组件
### 依赖赖组件：node-exporter、cAdvisor、Prometheus、Grafana
#### node-exporter
node-export 主要主要是监控kubernetes 集群node 物理主机：cpu、memory、network、disk 等基础监控资源。
使用daemonset 方式 自动为每个node部署监控agent。
#### cAdvisor
cAdvisor是Kubernetes内置的一个基础的监控Kubernetes下各种不同维度（Containers/Pods/Services/Cluster）指标的组件。
#### kube-state-metrics
kube-state-metrics是一种简单的服务，它监听Kubernetes API服务器并生成有关对象状态的度量标准。它不关注单个Kubernetes组件的健康状况，而是关注内部各种对象（如部署，节点和Pod）的健康状况。
####  Prometheus
Prometheus是继Kubernetes之后第二个加入Cloud Native Computing Foundation的托管项目，是一套开源的监控告警工具包，他能够收集来自不同node暴露出来的metrics并且通过PromQL实现指标过滤、查询等功能。
#### Grafana
Grafana是一套功能丰富的指标仪表板和数据图表编辑器，它可以处理来自Prometheus的数据源，并且提供多种可视化的数据表示功能。

## 2. 架构图
![架构图](/doc/frame.jpg)

通过node-exporter收集node相关的监控指标，通过cAdvisor和kub-state-metics收集pod，container维度相关的指标，并通过prometheus收集整合各个node上收集好的数据，可以通过promeQL语句对收集的指标进行筛选、聚合。最后，借助grafana进行对指标的可视化。

## 3. yaml文件介绍
![目录图](/doc/content.png)

grafana/grafana-service.yaml：创建grafana service，提供外网访问的loadbalancer IP，端口为8081<br>
grafana/grafana-deployment.yaml：部署grafana<br>
grafana/\*-config.yaml: grafana相关的配置文件，包含datasource和dashboard
kubernetes/kube-state-metrics-service-account.yaml：kube-state-metrics需要获取每个node上kubernetes的各项指标，所以需要为其创建一个service account，同时下面需要创建role和role-binding<br>
kube-state-metrics/kube-state-metrics-service：创建kube-state-metrics服务<br>
node-exporter/node-exporter.yaml： 创建一个node-expoter的DaemonSet，确保全部Node上运行一个node-expoter的Pod副本<br>
prometheus/prometheus-service-account.yaml：promtheus需要获取node-exporter，kube-state-metrics以及cAdvisor暴露出来的metircs，所以需要为其创建一个service-account，同时下面需要创建role和role-binding<br>
prometheus/prometheus-config.yaml：创建一个configMap，用于prometheus的配置文件<br>
prometheus/prometheus-pvc.yaml： 创建一个persistent volume claim，因为prometheus的数据需要保存在CBS上。<br>
prometheus/prometheus-depolyment.yaml：部署prometheus<br>
prometheus/prometheus-service.yaml：创建prometheus service，使用lb，暴露端口为8081。
alertmanager/\*.yaml：部署普罗米修斯告警器
receiver/\*.yaml: 云监控自定义报警webhook的代理


## 4. 部署

	# kubectl create ns monitoring
	# kubectl create -f yaml-files/grafana -f yaml-files/kube-state-metrics -f yaml-files/node-exporter -f yaml-files/prometheus -f yaml-files/alertmanager -f yaml-files/receiver -n monitoring
![输出](/doc/output.png)

## 5. Prometheus

### 5.1 查看Prometheus控制台
1.在终端获取PrometheusLoadBalancerIP，端口默认为**8081**
>  # kubectl get svc/prometheus -o=jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}' -n monitoring

example:
![获取Prometheus控制台ip](/doc/get-prometheus-ip.png)

2.访问Prometheus控制台，检查Tagets是否已经全部up。
![Prometheus控制台](/doc/prometheus-console.png)

### 5.2 Prometheus基本查询语法
Prometheus提供一个函数式的表达式语言，可以使用户实时地查找和聚合时间序列数据。表达式计算结果可以在图表中展示，也可以在Prometheus表达式浏览器中以表格形式展示，或者作为数据源, 以HTTP API的方式提供给外部系统使用。

#### 5.2.1 表达式语言数据类型
在Prometheus的表达式语言中，任何表达式或者子表达式都可以归为四种类型：

	1. 即时向量(instant vector) 包含每个时间序列的单个样本的一组时间序列，共享相同的时间戳。
	2. 范围向量(Range vector) 包含每个时间序列随时间变化的数据点的一组时间序列。
	3. 标量(Scalar) 一个简单的数字浮点值
	4. 字符串(String) 一个简单的字符串值(目前未被使用)

examples:<br>
查询K8S集群内所有apiserver健康状态
> (sum(up{job="apiserver"} == 1) / count(up{job="apiserver"})) * 100 <br>

查询pod 聚合一分钟之内的cpu 负载
> sum by (container_name)(rate(container_cpu_usage_seconds_total{image!="",container_name!="POD",pod_name="XXXX"}[1m]))

#### 5.2.2 时间序列选择器
##### 即时向量选择器
即时向量选择器允许选择一组时间序列，或者某个给定的时间戳的样本数据。下面这个例子选择了具有container_cpu_usage_seconds_total的时间序列：<br>
>  container_cpu_usage_seconds_total

你可以通过附加一组标签，并用{}括起来，来进一步筛选这些时间序列。下面这个例子只选择有container_cpu_usage_seconds_total名称的、有prometheus工作标签的、有pod_name组标签的时间序列：<br>

> container_cpu_usage_seconds_total{image!="",container_name!="POD",pod_name="XXXXXX"}

另外，也可以也可以将标签值反向匹配，或者对正则表达式匹配标签值。下面列举匹配操作符：
> =：选择正好相等的字符串标签<br>
> !=：选择不相等的字符串标签<br>
> =~：选择匹配正则表达式的标签（或子标签）<br>
> !=：选择不匹配正则表达式的标签（或子标签）<br>

例如，选择staging、testing、development环境下的，GET之外的HTTP方法的http_requests_total的时间序列：
> http_requests_total{environment=~"staging|testing|development",method!="GET"}

##### 范围向量选择器
范围向量表达式正如即时向量表达式一样运行，前者返回从当前时刻的时间序列回来。语法是，在一个向量表达式之后添加[]来表示时间范围，持续时间用数字表示，后接下面单元之一：

时间长度有一个数值决定，后面可以跟下面的单位：
```
s - seconds
m - minutes
h - hours
d - days
w - weeks
y - years
```
在下面这个例子中，我们选择此刻开始1分钟内的所有记录，metric名称为container_cpu_usage_seconds_total、作业标签为pod_name的时间序列的所有值：
> irate(container_cpu_usage_seconds_total{image!="",container_name!="POD",pod_name="acw62egvxd95l7t3q5uxee"}[1m])

##### 偏移修饰符(offset modifier)
偏移修饰符允许更改查询中单个即时向量和范围向量的时间偏移量，例如，以下表达式返回相对于当前查询时间5分钟前的container_cpu_usage_seconds_total值：
> container_cpu_usage_seconds_total offset 5m

如下是范围向量的相同样本。这返回container_cpu_usage_seconds_total在一天前5分钟内的速率：
> (rate(container_cpu_usage_seconds_total{pod_name="acw62egvxd95l7t3q5uxee"} [5m] offset 1d))  

##### 操作符
Prometheus支持多种二元和聚合的操作符[请查看这里](https://prometheus.io/docs/prometheus/latest/querying/operators/)

##### 函数
Prometheus支持多种函数，来对数据进行操作[请查看这里](https://prometheus.io/docs/prometheus/latest/querying/functions/)


## 6. Grafana

### 6.1 登录Grafana控制台
在终端获取GrafanaLoadBalancerIP，端口默认为**8081**，用户名：**admin**，默认密码为**dangerous**
>  # kubectl get svc/grafana -o=jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}' -n monitoring

example:
![获取Grafana控制台ip](/doc/get-grafana-ip.png)
### 6.2 效果图
![最终结果](/doc/final.png)

![最终结果](/doc/final-1.png)

## 7. 参考链接

1. prometheus官网：[https://prometheus.io/docs/introduction/overview/](https://prometheus.io/docs/introduction/overview/)
2. prometheus语法学习：[https://prometheus.io/docs/prometheus/latest/querying/basics/](https://prometheus.io/docs/prometheus/latest/querying/basics/) 
3. grafana官网：[http://docs.grafana.org/](http://docs.grafana.org/)
4. grafana dashboard模板交流社区：[https://grafana.com/dashboards](https://grafana.com/dashboards)
5. 搭建kubernetes监控文章：[https://www.kancloud.cn/huyipow/prometheus](https://www.kancloud.cn/huyipow/prometheus)