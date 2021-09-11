---
title: Prometheus
date: 2021-09-05 09:54:20
tags: [prometheus,centos]
---

> Prometheusis an open-source systems monitoring and alerting toolkit originally built at SoundCloud. 
> 普罗米修斯是最初由SoundCloud建立的一套开源的系统监控及预警的工具。

[TOC]

### 一. 软件简介

Prometheus 与 Influxdb都是时序数据库，比起关系型数据库，它们在时间序列数据的处理上具有极大的优势，例如长期高频率记录温度传感器的数值，该数据跟时间关联较大，且数据量极大。

### 二. 安装 Prometheus

#### 1. 下载
```
# /opt 目录类似于Windows的C://Program Files/目录
cd /opt/
# 下载prometheus的压缩文件
wget https://github.com/prometheus/prometheus/releases/download/v2.29.1/prometheus-2.29.1.linux-amd64.tar.gz
# 解压
tar xzvf prometheus-2.29.1.linux-amd64.tar.gz
# 创建软链接，便于后续进行版本升级
ln -snf /opt/prometheus-2.29.1.linux-amd64.tar.gz /opt/prometheus
```

#### 2. Systemd
创建Service文件`vi /usr/lib/systemd/system/prometheus.service`：
```
[Unit]
Description=Prometheus
After=network.target # 依赖于网络服务

[Service]
Type=simple # 启动类型：ExecStart 创建的进程作为主进程
User=nobody # 执行用户使用 nobody
ExecStart=/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml --storage.tsdb.path=/mnt/var/prometheus --web.console.libraries=/opt/prometheus/console_libraries --web.console.templates=/opt/prometheus/consoles --web.enable-lifecycle # 启动 prometheus 进程
Restart=on-failure # 失败自动重启

[Install]
WantedBy=multi-user.target # 可以通过 systemctl enable 启用，将会将该服务包含到 multi-user.target 中，这样在启动 multi-user.target 时，将会自动启动 multi-user.target
```
> 使用Systemd的Service的好处在于，可以开机自启动，并且失败可以自动重启，简化了运维管理。
>
> 重新加载Service文件：`systemctl daemon-reload`
> 开机自启动：`systemctl enable prometheus`
> 启动服务：`systemctl start prometheus`
> 查看服务状态：`systemctl status prometheus`
> 查看服务输出：`journalctl -xe -u prometheus`

#### 3. 自带的图形化界面
访问 http://127.0.0.1:9090/graph

### 三. 安装 Exporter

> Prometheus 的官方与第三方提供了多种 Exporter，它们会采集各种监控数据，供Prometheus定期拉取（Pull）。

#### 1. Node Exporter（硬件与操作系统）
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
```
`vi /usr/lib/systemd/system/node_exporter.service`
```
[Unit]
Description=NodeExporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/opt/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
`systemctl daemon-reload & systemctl enable node_exporter & systemctl start node_exporter`

#### 2. Mysqld Exporter（MySQL服务）
```
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.13.0/mysqld_exporter-0.13.0.linux-amd64.tar.gz
```
`vi /usr/lib/systemd/system/mysqld_exporter.service`
```
[Unit]
Description=MysqldExporter
After=network.target

[Service]
Type=simple
User=nobody
Environment=DATA_SOURCE_NAME=user:password@(hostname:3306)/ # TODO Change
ExecStart=/opt/mysqld_exporter/mysqld_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
`systemctl daemon-reload & systemctl enable mysqld_exporter & systemctl start mysqld_exporter`

#### 3. Kafka Exporter
```
wget https://github.com/danielqsj/kafka_exporter/releases/download/v1.3.1/kafka_exporter-1.3.1.linux-amd64.tar.gz
```
`vi /usr/lib/systemd/system/kafka_exporter.service`
```
[Unit]
Description=KafkaExporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/opt/mysqld_exporter/kafka_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
`systemctl daemon-reload & systemctl enable kafka_exporter & systemctl start kafka_exporter`


### 四. Pull

> Prometheus 根据配置文件中的`scrape_configs`，自动定期拉从Exporter拉取数据。

#### 1. 修改配置文件
`vi /opt/prometheus/prometheus.yml`
```
# 全局配置
global:
  scrape_interval: 15s # 每隔15s从Exporter拉取一次数据（根据scrape_configs）
  evaluation_interval: 15s # 每隔15s计算一次规则（根据rule_files）

# Alertmanager相关配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# 规则文件列表
rule_files:

# 抓取配置列表
scrape_configs:
  - job_name: "prometheus" # 自带的Job，拉取Prometheus自身的数据
    static_configs:
      - targets: ["localhost:9090"]

## 以下为新增项

  - job_name: "nodes" # 拉取 Node Exporter 的数据
    static_configs:
      - targets: ["localhost:9100"] # Node Exporter 默认监听 9100 端口

  - job_name: "mysqld" # 拉取 Mysqld Exporter 的数据
    static_configs:
      - targets: ["localhost:9104"] # Node Exporter 默认监听 9104 端口

  - job_name: "kafka" # 拉取 Kafka Exporter 的数据
    static_configs:
      - targets: ["localhost:9308"] # Node Exporter 默认监听 9308 端口

  - job_name: "pushgateway"
    static_configs:
      - targets: ["localhost:9091"] # PushGateway 默认监听 9091 端口
```
#### 2. 要求Prometheus重新加载配置文件
```
# 如果未添加 -web.enable-lifecycle，这个接口会返回：Lifecycle API is not enabled
curl -X POST http://localhost:9090/-/reload
```


### 五. 更好的可视化工具：Grafana

#### 1. 安装并启动
```
wget https://dl.grafana.com/oss/release/grafana-8.1.1-1.x86_64.rpm
yum install grafana-8.1.1-1.x86_64.rpm

systemctl start grafana-server
```

#### 2. 初始管理员密码

访问 http://localhost:3000, 默认账户为 admin:admin， admin首次登陆需要重设密码。

#### 3. 数据源与模板

添加Prometheus为数据源，然后去Grafana的[Dashboards](https://grafana.com/grafana/dashboards?dataSource=prometheus "Dashboards")找一些合适的模板。

[1 Node Exporter for Prometheus Dashboard CN v20201010](https://grafana.com/grafana/dashboards/8919 "1 Node Exporter for Prometheus Dashboard CN v20201010")：
