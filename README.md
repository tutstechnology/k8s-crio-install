# Install Cluster K8S with CRI-O on Ubuntu 20.04
## Kubernetes (Master and Control Plane) and services: CRI-O, Dashboard, Metrics, Ambassador and Prometheus

-----------------------
### Information:

_This manual summarizes the good installation practices for the Kubernetes Cluster. It is the combination of several technical references that I found on the internet and I gathered them all in this manual._

_The installation simulation is based on a Master server and two Workers server nodes. For each step, there is information (master or worker) indicating where the configuration and deployment should be performed._

#### **Versions used:**

**Kubernetes:** *1.19.6* |
**CRI-O:** *1.19.0* |
**K8S Dashboard:** *2.1.0* |
**Metrics:** *0.4.1* |


**IMPORTANT:** You can modify the versions by changing the variables exemplified in this document.

-----------------------

**In this installation plan, we will use the following servers / ip:**

`hpv-prd-k8s-cplane01`     `192.168.15.11`   

`hpv-prd-k8s-worker01`     `192.168.15.12`   

`hpv-prd-k8s-worker02`     `192.168.15.13` 



-----------------------
### Port requirements for cluster services:

| service | master | worker | port type |
|:-:|:-:|:-:|:-:|
| kubelet  | 43925*  | 39305*  | dynamic  |
| kubelet  | 10250  | 10250  | fixed  |
| kubelet  | 10248  | 10248  | fixed |
| container  | 42835*  | 41425*  | dynamic |
| kube-sche  | 10259  |   | fixed  |
| kube-apis  | 6443  |   | fixed  |
| kube-cont  | 10257  |   | fixed  |
| kube-prox  | 10256  | 10256  | fixed  |
| kube-prox  | 10249  | 10249  | fixed  |
| kubectl  | 5100  |   | fixed  |

###### _*Example ports_

_This information is important for defining firewall rules between nodes._

###### After completing the installation using all the steps below, it is possible to consult the ports used. Run the command:
```
lsof -i -P -n | grep LISTEN
```

-----------------------
### Installation index:

**step 1 to step 5 -** _Requirements and settings for Linux - used (Ubuntu 20.04)._

**step 6            -** _CRI-O installation._

**step 7 to step 12 -** _Kuberntes installation._

**step 13 to step 17 -** _Dashboard installation._

**step 18 to step 19 -** _Metrics installation._

**step 20 to step 21 -** _Ambassador installation._

**step 22 to step 25 -** _Prometheus installation._

-----------------------


### Config / Linux Requirements:


#### **Step 1**: | Master | Worker | - _Update the latest packages using the apt `update command`:_
```
sudo apt update -y
```

#### **Step 2**: | Master | Worker | - _Define the name and ip of the servers that form the k8s cluster. Edit the `/etc/hosts` file and add the cluster servers:_
Example:
```
127.0.0.1 localhost
192.168.15.11   hpv-prd-k8s-cplane01
192.168.15.12   hpv-prd-k8s-worker01
192.168.15.13   hpv-prd-k8s-worker02
```

#### **Step 3**: | Master | Worker | - _Set the `hostname` for the Master server:_
Example:

Set hostname:
```
hostnamectl set-hostname "server name"
```

#### **Step 4**: | Master | Worker | - _Disable memory `SWAP`:_
```
sudo swapon -s
sudo swapoff -a
```

Comment if you have the Swap line on `fstab`:
```
sudo vim /etc/fstab
```
Exemple: **#**/swap.img      none    swap    sw      0       0



#### **Step 5**: | Master | Worker | - _Restart the server:_
```
sudo reboot
```
-----------------------
### Install CRI-O:

#### **Step 6**: | Master | Worker | - _Installing CRI-O:_

Load the overlay and br_netfilter modules:
```
sudo modprobe overlay
sudo modprobe br_netfilter
```

File creation `99-kubernetes-cri.conf`:
```
vim /etc/sysctl.d/99-kubernetes-cri.conf
```

Copy and paste the information below into the file:
```
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

Edit file `sysctl.conf`:
```
vim /etc/sysctl.conf
```

Copy and paste the information below into the file:
```
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
```


Apply the config file by running:
```
sudo sysctl --system
```

Create variables for Ubuntu amd CRI-O version:
```
export OS=xUbuntu_20.04
export VERSION=1.19
```

Add CRI-O repositories and install:
```
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt-get update
apt-get install cri-o cri-o-runc
```

Find Conmon's path to the next step:
```
which conmon
```
Command result example:
```
/usr/bin/conmon
```

Edit `/etc/crio/crio.conf`:
```
vim /etc/crio/crio.conf
```
_You must include the following lines in the file:_

_Modify in (line 104)_
```
conmon = "/usr/bin/conmon" 
```

_Insert to (line 365)_
```
registries = [
        "docker.io",
        "quay.io",
]
```

Enable the CRI-O and make sure that it is running:
```
sudo systemctl daemon-reload
sudo systemctl enable crio
sudo systemctl start crio
sudo systemctl status crio
```


-----------------------
### Installation and configuration Kubernetes:

#### **Step 7**: | Master | Worker | - _Install Kubernetes Packages (`kubeadm`, `kubelet`, `kubectl`):_

Add K8S repositories and install:
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
```

Create variable for Kubernetes version:
```
export K8SVERSION=1.19.6-00
```
**IMPORTANT:** You can find the specific version by the following command:
```
curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'
```

Install `Kubeadm`, `kubectl`, `kubelet`and `kubernetes-cni`
```
sudo apt install -y kubeadm=$K8SVERSION kubectl=$K8SVERSION kubelet=$K8SVERSION kubernetes-cni --allow-unauthenticated
```

File creation `kubelet`:
```
vim /etc/default/kubelet
```
Copy and paste the information below into the file:
```
KUBELET_EXTRA_ARGS=--feature-gates="AllAlpha=false,RunAsGroup=true" --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock' --runtime-request-timeout=5m
```


#### **Step 8**: | Master | - _Cluster K8S configuration. Definition of the invite and pod network:_
```
kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr=10.244.0.0/16 --service-dns-domain tutstechnology.local
```
**IMPORTANT:** The command above will start the cluster and then display the command line I need to execute on my other nodes.
NOTE the information to join the Workers servers. 
Command result example:
```
kubeadm join 10.130.200.25:6443 --token kqbyqy.q6543jyyx6xl84yd \
    --discovery-token-ca-cert-hash sha256:94a749272471966abeb39c7bb74a597603994e091cfda17a1915b1eb72625c2c
```

#### **Step 9**: | Master | - _Cluster K8S configuration:_
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### **Step 10**: | Master | - _Cluster K8S configuration. Definition of the CNI `network component` for Pods Network add-on on Kuberntes:_

**IMPORTANT:** Choose only one of the models below:

Model 1 - Flannel
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

Model 2 - Weave Net
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Model 3 - Calico
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```



List podsnetwork:
```
kubectl get pods -n kube-system
```

#### **Step 11**: | Worker | - _Join Worker node to the Cluster:_

**IMPORTANT:** Use the command output seen in **step 8**
```
kubeadm join 10.130.200.25:6443 --token kqbyqy.q6543jyyx6xl84yd \
    --discovery-token-ca-cert-hash sha256:94a749272471966abeb39c7bb74a597603994e091cfda17a1915b1eb72625c2c
```


If you have not noted the Join command, execute the command below to obtain it:

```
kubeadm token create --print-join-command
```


#### **Step 12**: | Master | - _Commands to check the nodes/pods:_
```
kubectl get nodes
kubectl get pods --all-namespaces
```
-----------------------
### Installation and configuration Dashboard (v2.0.3):

#### **Step 13**: | Master | - _Dashboard Deploy:_

Oficial Repository:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
```

#### **Step 14**: | Master | - _Dashboard Deploy - Create `Admin User`:_

File creation (1) `createuseradmin-user.yaml`:
```
nano createuseradmin-user.yaml
```
Copy and paste the information below into the file:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

File creation (2) `dashboard-adminuser.yaml`:
```
nano dashboard-adminuser.yaml
```
Copy and paste the information below into the file:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Offline Deploy:
```
kubectl apply -f [local directory] ../createuseradmin-user.yaml
kubectl apply -f [local directory] ../dashboard-adminuser.yaml
```

#### **Step 15**: | Master | - _Dashboard Deploy - Getting a Bearer `Token`:_

**IMPORTANT:** The command below will generate a token, make a NOTE to use in the console login.

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

Command result example:
```
Name:         admin-user-token-j8wqq
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 7e03d604-e4f7-4925-85dc-cbd9feb1ff51

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:    
eyJhbGciOiJSUzI1NiIsImtpZCI6ImlKeTdwWHhyMnNUOHZ3MVpuN0JqRnVQUWVSajNEOUM4bUtyNnRMNXBGZGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWo4d3FxIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3ZTAzZDYwNC1lNGY3LTQ5MjUtODVkYy1jYmQ5ZmViMWZmNTEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.WVWY9SQAqHaiNF8yoJnpjVmXIBBPbkxksrwwh30eTq370gCObR5hsCmFBG60zPUgfehKZ9IFFFAkgcB0m2JAid8IAN1MaQQRuTp1VYg7z-xjqvC1U9xh3t-7k519EVq9AER_VtDiqh0GRVHRw5qa8jLG7ObYua6PyZn_cz6VvSOTT3bsK_GQHa5UCCW9P2suyZm4B4ztdtjCvpN7kQBFo5Mf9EmxCV5lsKI5EpBthRpniPItP-haKLyW14acJK3Mocuwg09cVOwcKeMaqhoMLNEIyn35o3EOTuG6xDVQFqVsnfateiRaN_Wza63cQ8UZtWrhfFi5MxgpwbfnHyHPUA
root@aws-prd-k8smaster01:/etc/kubernetes/manifests/tutstechnology/dashboard/v203#
```

Below is the token that should be noted:

`eyJhbGciOiJSUzI1NiIsImtpZCI6ImlKeTdwWHhyMnNUOHZ3MVpuN0JqRnVQUWVSajNEOUM4bUtyNnRMNXBGZGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWo4d3FxIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3ZTAzZDYwNC1lNGY3LTQ5MjUtODVkYy1jYmQ5ZmViMWZmNTEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.WVWY9SQAqHaiNF8yoJnpjVmXIBBPbkxksrwwh30eTq370gCObR5hsCmFBG60zPUgfehKZ9IFFFAkgcB0m2JAid8IAN1MaQQRuTp1VYg7z-xjqvC1U9xh3t-7k519EVq9AER_VtDiqh0GRVHRw5qa8jLG7ObYua6PyZn_cz6VvSOTT3bsK_GQHa5UCCW9P2suyZm4B4ztdtjCvpN7kQBFo5Mf9EmxCV5lsKI5EpBthRpniPItP-haKLyW14acJK3Mocuwg09cVOwcKeMaqhoMLNEIyn35o3EOTuG6xDVQFqVsnfateiRaN_Wza63cQ8UZtWrhfFi5MxgpwbfnHyHPUA`
 


#### **Step 16**: | Master | - _Dashboard - Use Proxy:_
```
kubectl proxy --address='0.0.0.0' --port=5100 --accept-hosts='.*' &
```

**IMPORTANT:** To access the console of a remote computer, use the command to route ports through SSH. See how in the next step.


#### **Step 17**: | Client | - _Dashboard - Console access:_

_Use this command on the customer's computer, which will access a web console (
Example for server ip `10.130.200.25`):_

```
ssh -L 5888:127.0.0.1:5100 10.130.200.25
```
_or (with Access Key):_
```
ssh -i mykey.pem -L 5888:127.0.0.1:5100 ubuntu@10.130.200.25
```

Now, access the console using the following address:
```
http://localhost:5888/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
**IMPORTANT:** Use the Token generated in step 15.


-----------------------
### Installation and configuration Metrics (v0.3.7):

#### **Step 18**: | Master |- _Metrics Deploy:_

Oficial Repository:
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.1/components.yaml
```

```
kubectl get apiservices |egrep metrics
kubectl get deploy,svc -n kube-system |egrep metrics-server
```


#### **Step 19**: | Master | - Metrics Deploy - _Modify Parameters:_


**IMPORTANT:** Depending on your cluster setup, you may also need to change flags passed to the Metrics Server container. Most useful flags:

Use:

_**--kubelet-insecure-tls**      o not verify CA of serving certificates presented by Kubelets.  For testing purposes only.
**--kubelet-preferred-address-types strings**        The priority of node address types to use when determining which address to use to connect to a particular node (default [Hostname,InternalDNS,InternalIP,ExternalDNS,ExternalIP])_

```
kubectl -n kube-system edit deploy metrics-server
```
_You must include the following lines in the file.:_
```
       command:
       - /metrics-server
       - --kubelet-insecure-tls
       - --kubelet-preferred-address-types=InternalIP
```

Example:
_insert after line 157_
```
...
spec:
     containers:
     - args:
       - --cert-dir=/tmp
       - --secure-port=4443
       image: k8s.gcr.io/metrics-server/metrics-server:v0.3.7
       command:
       - /metrics-server
       - --kubelet-insecure-tls
       - --kubelet-preferred-address-types=InternalIP
       imagePullPolicy: IfNotPresent
       name: metrics-server
       ports:
       - containerPort: 4443
         name: main-port
         protocol: TCP
        resources: {}
...
```

To verify that the changes were made to the file, run the command:
```
kubectl get deploy metrics-server -n kube-system -o yaml | grep command -A 4
```

Need to restart the `kubelet` service:
```
systemctl restart kubelet
systemctl enable kubelet
```

After this modification, the `kubectl top` command can now be used without errors.

Examples:
```
kubectl top nodes
kubectl top pods --all-namespaces
kubectl top pods --all-namespaces --sort-by=memory
```

------
### Installation Ambassador Edge Stack:

#### **Step 20**: | Master | - _Ambassador Deploy:_
```
kubectl apply -f https://www.getambassador.io/yaml/aes-crds.yaml && \
kubectl wait --for condition=established --timeout=90s crd -lproduct=aes && \
kubectl apply -f https://www.getambassador.io/yaml/aes.yaml && \
kubectl -n ambassador wait --for condition=available --timeout=90s deploy -lproduct=aes
```

#### **Step 21**: | Master | - _Ambassador Deploy - Determine the IP address or hostname of your cluster:_
```
kubectl get -n ambassador service ambassador -o "go-template={{range .status.loadBalancer.ingress}}{{or .ip .hostname}}{{end}}"
```

------
### Installation Prometheus:

#### **Step 22**: | Master | - _Prometheus Deploy - Create a Namespace:_

```
kubectl create namespace monitoring
```

#### **Step 23**: | Master | - _Prometheus Deploy - Create a Files:_

**File creation (1) - Cluster Role** `clusterRole.yaml`:

```
nano clusterRole.yaml
```
Copy and paste the information below into the file:
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```


**File creation (2) - Config Map** `config-map.yaml`:

```
nano config-map.yaml
```
Copy and paste the information below into the file:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: devopscube demo alert
      rules:
      - alert: High Pod Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
      
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      
      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
```

**File creation (3) - Prometheus Deployments** `prometheus-deployment.yaml`:

```
nano prometheus-deployment.yaml
```
Copy and paste the information below into the file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
  
        - name: prometheus-storage-volume
          emptyDir: {}
```

**File creation (4) - Prometheus as a Service** `prometheus-service.yaml`:

```
nano prometheus-service.yaml
```
Copy and paste the information below into the file:
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus-server
  type: NodePort  
  ports:
    - port: 8080
      targetPort: 9090 
      nodePort: 30000
```


#### **Step 24**: | Master | - _Prometheus Deploy - Deploy Files:_

Offline Deploy:
```
kubectl create -f [local directory] ../clusterRole.yaml
kubectl create -f [local directory] ../config-map.yaml
kubectl create -f [local directory] ../prometheus-deployment.yaml
kubectl create -f [local directory] ../prometheus-service.yaml --namespace=monitoring
```

#### **Step 25**: | Master | - _Prometheus Deploy - Verify:_

```
kubectl get deployments --namespace=monitoring
kubectl get pods --namespace=monitoring
```




