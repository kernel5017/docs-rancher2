---
title: CA证书和4层负载均衡的cluster.yml 文件模板
description: RKE 使用 cluster.yml 文件安装和配置您的 Kubernetes 集群。如果您使用配置是CA证书和4层负载均衡，您可以使用这个 cluster.yml 模板安装和配置集群。
keywords:
  - rancher 2.0中文文档
  - rancher 2.x 中文文档
  - rancher中文
  - rancher 2.0中文
  - rancher2
  - rancher教程
  - rancher中国
  - rancher 2.0
  - rancher2.0 中文教程
  - 安装指南
  - 资料、参考和高级选项
  - cluster.yml 文件模板
  - CA证书和4层负载均衡的cluster.yml 文件模板
---

RKE 使用 cluster.yml 文件安装和配置您的 Kubernetes 集群。

如果您使用配置如下所示，您可以使用这个 cluster.yml 模板安装和配置集群。

- CA 签发的证书
- 4 层负载均衡
- [NGINX Ingress controller](https://kubernetes.github.io/ingress-nginx/)

详情请参考[RKE 文档](/docs/rke/config-options/_index)

```yaml
nodes:
  - address: <IP> # hostname or IP to access nodes
    user: <USER> # root user (usually 'root')
    role: [controlplane, etcd, worker] # K8s roles for node
    ssh_key_path: <PEM_FILE> # path to PEM file
  - address: <IP>
    user: <USER>
    role: [controlplane, etcd, worker]
    ssh_key_path: <PEM_FILE>
  - address: <IP>
    user: <USER>
    role: [controlplane, etcd, worker]
    ssh_key_path: <PEM_FILE>
services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h
addons: |-
  ---
  kind: Namespace
  apiVersion: v1
  metadata:
    name: cattle-system
  ---
  kind: ServiceAccount
  apiVersion: v1
  metadata:
    name: cattle-admin
    namespace: cattle-system
  ---
  kind: ClusterRoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: cattle-crb
    namespace: cattle-system
  subjects:
  - kind: ServiceAccount
    name: cattle-admin
    namespace: cattle-system
  roleRef:
    kind: ClusterRole
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: cattle-keys-ingress
    namespace: cattle-system
  type: Opaque
  data:
    tls.crt: <BASE64_CRT>  # ssl cert for ingress. If self-signed, must be signed by same CA as cattle server
    tls.key: <BASE64_KEY>  # ssl key for ingress. If self-signed, must be signed by same CA as cattle server
  ---
  apiVersion: v1
  kind: Service
  metadata:
    namespace: cattle-system
    name: cattle-service
    labels:
      app: cattle
  spec:
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
    selector:
      app: cattle
  ---
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    namespace: cattle-system
    name: cattle-ingress-http
    annotations:
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"   # Max time in seconds for ws to remain shell window open
      nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"   # Max time in seconds for ws to remain shell window open
  spec:
    rules:
    - host: <FQDN>  # FQDN to access cattle server
      http:
        paths:
        - backend:
            serviceName: cattle-service
            servicePort: 80
    tls:
    - secretName: cattle-keys-ingress
      hosts:
      - <FQDN>      # FQDN to access cattle server
  ---
  kind: Deployment
  apiVersion: extensions/v1beta1
  metadata:
    namespace: cattle-system
    name: cattle
  spec:
    replicas: 1
    template:
      metadata:
        labels:
          app: cattle
      spec:
        serviceAccountName: cattle-admin
        containers:
        # Rancher install via RKE addons is only supported up to v2.0.8
        - image: rancher/rancher:v2.0.8
          args:
          - --no-cacerts
          imagePullPolicy: Always
          name: cattle-server
  #       env:
  #       - name: HTTP_PROXY
  #         value: "http://your_proxy_address:port"
  #       - name: HTTPS_PROXY
  #         value: "http://your_proxy_address:port"
  #       - name: NO_PROXY
  #         value: "localhost,127.0.0.1,0.0.0.0,10.43.0.0/16,your_network_ranges_that_dont_need_proxy_to_access"
          livenessProbe:
            httpGet:
              path: /ping
              port: 80
            initialDelaySeconds: 60
            periodSeconds: 60
          readinessProbe:
            httpGet:
              path: /ping
              port: 80
            initialDelaySeconds: 20
            periodSeconds: 10
          ports:
          - containerPort: 80
            protocol: TCP
          - containerPort: 443
            protocol: TCP
```
