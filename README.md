# 1.部署grafana-prometheus-alertmanager到kubernetes

## 1.1版本说明

- grafana v7.2.0
- grafana github地址https://github.com/grafana/grafana/tree/v7.2.0
- grafana官方文档地址https://grafana.com/docs/grafana
- prometheus v2.21.0
- prometheus github地址https://github.com/prometheus/prometheus
- prometheus官方文档地址https://prometheus.io/docs/prometheus
- kubernetes版本1.18.3

## 1.2镜像版本说明

kubernetes集群可访问外网的时候，可直接使用以下镜像；当集群不可访问外网的情况，下载以下镜像上传到集群可以访问的私有镜像仓库。

- grafana/grafana:7.2.0
- busybox:latest
- prom/alertmanager:v0.21.0
-  prom/prometheus:v2.21.0
-  prom/node-exporter:v1.0.1
-  bitnami/kube-state-metrics:1.9.7
-  prom/blackbox-exporter:v0.17.0

## 1.3描述文件说明

描述文件下载地址https://github.com/yinchongbing/grafana-prometheus-alertmanager

### 1.3.1创建步骤

```sh
kubectl apply -f namespace.yaml
kubectl apply -f alertmanager-template-cm.yaml
kubectl apply -f prometheus-deploy.yaml
kubectl apply -f grafana-deploy.yaml
```

### 1.3.2关于持久化pvc说明

由于当前集群接入cephfs，因此有存储类（storageclass）cephfs；如果没有请使用其他持久化存储或者去掉持久化存储或者换成自己的存储类

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-storage
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: cephfs      #这里需要更改
  volumeMode: Filesystem
---
```

### 1.3.3关于grafana初始化密钥

这里的username和password都是admin，只因经过了base64加密

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana
  namespace: monitoring
data:
  admin-password: YWRtaW4=   #注意这里
  admin-username: YWRtaW4=   #注意这里
type: Opaque
```

### 1.3.4关于grafana目录无权限问题说明

grafana5.1版本后 groupid 更改了引起的问题，需要更改目录所属用户；

如果没做持久化存储，不需要关心这个问题。

如果做了持久化存储，在initContainers赋予权限即可。

grafana-deploy.yaml

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      task: monitoring
      k8s-app: grafana
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      containers:
        - name: grafana
          image: 10.110.63.25/rancher/grafana:7.2.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          env:
            - name: GF_AUTH_BASIC_ENABLED
              value: 'true'
            - name: GF_AUTH_ANONYMOUS_ENABLED
              value: 'true'
            - name: GF_AUTH_ANONYMOUS_ORG_ROLE
              value: Admin
            - name: GF_DASHBOARDS_JSON_ENABLED
              value: 'true'
            - name: GF_INSTALL_PLUGINS
              value: grafana-kubernetes-app
            - name: GF_SECURITY_ALLOW_EMBEDDING
              value: 'true'
            - name: GF_SECURITY_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  key: admin-username
                  name: grafana
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: admin-password
                  name: grafana
          volumeMounts:
          - mountPath: /var/lib/grafana
            name: grafana-storage
      initContainers:
        - name: grafana-chown
          command: ["chown", "-R", "472:472", "/var/lib/grafana"]
          image: 10.110.63.25/rancher/busybox:latest
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: grafana-storage
              mountPath: /var/lib/grafana
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana-storage
```

### 1.3.5关于prometheus rbac认证访问

如果此前在集群其他命名空间安装过prometheus，可能会存在名字叫prometheus的ClusterRole资源和ClusterRoleBindding资源；为了不引起冲突，可在创建prometheus的ServiceAccount、ClusterRole、ClusterRoleBindding的时候以命名空间为后缀，比如prometheus-ipaas如下yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-ipaas
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-ipaas
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-ipaas
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-ipaas
subjects:
  - kind: ServiceAccount
    name: prometheus-ipaas
    namespace: monitoring
```

### 1.3.6关于prometheus自动发现自己部署到集群的服务

本次部署的prometheus未配置自动发现自己的服务。

可以参照下面kubernetes-services这个job的配置适当更改；这个配置采集的是kubernetes的service，采集pod同样的道理。

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
# - "first_rules.yml"
# - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

      # metrics_path defaults to '/metrics'
      # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  #以下是kubernetes动态配置prometheus采集端点
  - job_name: kubernetes-services  #job名称，自定义
    kubernetes_sd_configs:
    - role: service                #只能是一下几种:pod,service,endpoint,ingress,node;详情见官方文档https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config
#      namespaces:                 #namespaces可以指定采集端点的名称空间，names下是数组，应该可以指定多个名称空间；不指定表示采集所有的名称空间
#        names:
#          - ingp-ipaas
    metrics_path: /actuator/prometheus     #指标采集url
    relabel_configs:
    - source_labels: [__meta_kubernetes_service_label_application]  #这里过滤kubernetes中带有appplication标签的service
      action: keep
      regex: ingp-ipaas                                             #过滤出application=ingp-ipaas的service
#    - source_labels: [__meta_kubernetes_service_name,__meta_kubernetes_namespace]
#      action: replace
#      regex: (.*);(.*)
#      replacement: ${1}.${2}
#      target_label: __address__
    - action: labelmap                                         #这里action和regex组合，表示把service的标签合并到采集的端点里
      regex: __meta_kubernetes_service_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]             #这里是给端点打上标签kubernetes_namespace='source_labels获取的值'
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]          #这里是给端点打上标签kubernetes_service_name='source_labels获取的值'
      target_label: kubernetes_service_name
```

