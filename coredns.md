# CoreDNS

## CoreDNS 简介

CoreDNS 是 [CNCF](https://www.cncf.io/) 的顶级项目之一。


## 部署 CoreDNS

### 修改配置

```bash
$ wget -O coredns.yaml https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.9/cluster/addons/dns/coredns.yaml.base

# 设置 dns 根域
$ sed -i "s|__PILLAR__DNS__DOMAIN__|cluster.local|g" coredns.yaml

# 设置 Service CIDR 和 Pod CIDR（参考： https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed#L53）
$ sed -i "s|__PILLAR__CLUSTER_CIDR__|172.254.0.0/16 172.1.0.0/16|g" coredns.yaml

# 修改 coredns ServiceIP，需要与 kubelet 保持一致
$ sed -i "s|__PILLAR__DNS__SERVER__|172.254.0.2|g" coredns.yaml
```

另外，默认的 coredns.yaml 文件设置了 `node-role.kubernetes.io=master:NoSchedule` toleration，所以 coredns 可以会被部署到 Master 节点上，如果不想部署到 Master 节点可以删除该 toleration。

### 部署及检查

部署 CoreDNS 前先停掉 kube-dns：

```bash
$ kubectl -n kube-system delete deploy/kube-dns
$ kubectl -n kube-system delete service/kube-dns
$ kubectl -n kube-system delete configmap/kube-dns
$ kubectl -n kube-system delete serviceaccount/kube-dns
```

```bash
# 部署 coredns
$ kubectl apply -f coredns.yaml

# 检查部署的服务
$ kubectl -n kube-system get svc,deploy,pod -l k8s-app=coredns
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
svc/coredns   ClusterIP   172.254.0.2   <none>        53/UDP,53/TCP,9153/TCP   37m

NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/coredns   1         1         1            1           38m

NAME                          READY     STATUS    RESTARTS   AGE
po/coredns-7869cf7cf7-xn4js   1/1       Running   0          38m

# 查看集群信息
$ kubectl cluster-info
Kubernetes master is running at https://192.168.10.80:6443
KubeDNS is running at https://192.168.10.80:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

# 排查日志
$ kubectl -n kube-system logs -f coredns-7869cf7cf7-xn4js
```

### 测试 CoreDNS

```bash
$ kubectl run nginx --image=nginx:alpine --port=80 --replicas=10
$ kubectl expose deploy/nginx --name=nginx --port=80 --target-port=80

$ kubectl get service nginx
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP   172.254.202.52   <none>        80/TCP    3d

# 测试所有 Pod 是否都可以正常解析
$ pods=`kubectl get pod -o wide | awk '{print $1}'`
$ for pod in $pods; do kubectl exec -it $pod -- nslookup nginx.default.svc.cluster.local; done

# 测试外网域名是否可以正常解析
$ for pod in $pods; do kubectl exec -it $pod -- nslookup baidu.com; done

# 移除测试应用
$ kubectl delete deploy/nginx
$ kubectl delete service/nginx
```

### coredns.yaml

最终生成的 coredns.yaml 文件内容如下：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        log stdout
        health
        kubernetes cluster.local 172.254.0.0/16 172.1.0.0/16
        prometheus
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:0.9.10
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 172.254.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```


## Systemd 运行 coredns

```bash
$ cat <<EOF > /usr/lib/systemd/system/coredns.service
[Unit]
Description=CoreDNS DNS server
Documentation=https://coredns.io
After=network.target

[Service]
PermissionsStartOnly=true
LimitNOFILE=1048576
LimitNPROC=512
CapabilityBoundingSet=CAP_NET_BIND_SERVICE 
AmbientCapabilities=CAP_NET_BIND_SERVICE 
NoNewPrivileges=true 
User=coredns
WorkingDirectory=~
ExecStart=/usr/bin/coredns -conf=/etc/coredns/Corefile
ExecReload=/bin/kill -SIGUSR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```


## 参考

* [coredns/deployment](https://github.com/coredns/deployment/tree/master/kubernetes)
* [coredns.service](https://github.com/coredns/deployment/blob/master/systemd/coredns.service)
