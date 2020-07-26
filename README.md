# cluster_deploy_ansible
1. 根据Hosts文件配置，在Agent节点部署对应服务
2. 服务包括Prometheus监控报警展示体系服务端(Prometheus+Grafana+alertmanager)，客户端(Node，Springboot服务，中间件服务等...)
3. 各个服务安装包需要去官网下载放在对应目录，其它如Grafana展示界面数据，Alertmanager报警指标文件则存在

# 设计原则

> 注：使用着需要了解Prometheus体系和Ansible，然后根据版本修改Ansible Playbook进行对应安装

- 初步实现持续集成流程中`部署/交付管理`的部分功能
- 初步实现持续集成流程中`环境管理`的`监控报警`和`数据可视化`的功能
- `一切配置可自由修改`

# 使用方法
> 注：使用前确定主节点已完成无密码访问，Ansible能正常连接Agent

- Hosts配置

```
# Prometheus 主节点
[pro-server]
#192.168.1.101 name=pro-monitor001

# Node_exporter 节点
[pro-client]
#192.168.1.101 name=pro-monitor001

[pro-all:children]
pro-server
pro-client

# Mysql_exporter 节点
[exporter-mysql]
#192.168.1.101 name=pro-monitor001
...
# Prometheus 代理节点
[pro-pushgateway]
#192.168.1.101 name=pro-monitor001

[pro_pushgateway_client]
#192.168.1.101 name=pro-monitor001
...
# Kibana安装节点
[kibana_node]
#192.168.1.101 name=pro-monitor001

# 全局变量
[all:vars]
VIP=192.168.1.101
...
```
## 配置文件修改
- 各个服务的service启动文件需要经过对应修改，启动文件在对应roles目录的file目录下，以Prometheus server为例
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
# 启动文件
--config.file /etc/prometheus/prometheus.yml \
# 数据存储路径
--storage.tsdb.path /var/lib/prometheus/ \
# 数据保存过期时间
--storage.tsdb.retention.time=7d \
# 日志级别
--log.level=warn \
--web.enable-lifecycle \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

## 安装执行
- Prometheus所属节点服务
```
# pro-install.yaml
---
# 所有Prometheus所属节点
- hosts: pro-all
  gather_facts: no
  roles:
  - { role: node_install }
# Prometheus Server节点
- hosts: pro-server
 roles:
 - { role: hosts_file }
 - { role: server_install }
 - { role: grafana_install }
 - { role: alertmanager_install }
# 根据配置安装监控server节点和Agent节点服务
ansible-playbook pro-install.yaml
```
- 中间件exporter和单独服务节点
```
# exporter-install.yaml
---
# exporter 节点　
- hosts: exporter-mysql
 roles:
 - { role: mysql_install }
# 代理节点
- hosts: pro-pushgateway
 roles:
 - { role: pushgateway_install }
- hosts: pro_pushgateway_client
 roles:
 - { role: pushgateway_client_install }
...
# 单独服务节点
- hosts: kibana_node
 roles:
 - { role: kibana_install }
# 网络存活性客户端节点
- hosts: exporter-blackbox
 roles:
 - { role: blackbox_install }
# 自定义exporter节点
- hosts: exporter-scripts
  roles:
  - { role: scripts_install }
...
# 根据配置安装节点服务
ansible-playbook exporter-install.yaml
```

# 其它
## Prometheus
- Server 配置
Prometheus Server安装完成之后，需要根据Exporter节点的启动端口，配置`prometheus.yml`
```
# 默认监控节点刷新时间
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus_master'
    scrape_interval: 5s
    static_configs:
    # Server节点端口
      - targets: ['prometheus_master:9090']
  - job_name: 'prometheus_node'
    scrape_interval: 5s
    static_configs:
    # Node端口
      - targets: [':prometheus_node9100']
  - job_name: 'prometheus_tomcat'
    scrape_interval: 5s
    static_configs:
    # 监控中间件服务端口
      - targets: ['prometheus_tomcat:8100']
  - job_name: 'pushgateway'
    scrape_interval: 5s
    static_configs:
    # 代理服务监控端口，公网地址在安装时请修改
      - targets: ['XXX.XXX.XXX.XXX:9091']
    honor_labels: true
  - job_name: 'http_check'
    metrics_path: /probe
    params:
      module: [http_2xx]  
    static_configs:
      - targets:
    #网站存活性监控网址地址
        - http://192.168.1.101:1234/actuator/info

    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: cloud-monitor002:9115  


alerting:
  alertmanagers:
  - static_configs:
  # 报警节点地址端口
    - targets: [ 'prometheus_master:9193' ]

rule_files:
# 监控报警指标目录，可根据情况自由配置
  - /etc/prometheus/rules/*.rules
```
- Prometheus 热加载功能
```
curl -X POST http://localhost:9090/-/reload
```

## 
