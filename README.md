# Workshop

Example for Centos.

## 1. Install kubernetes

### Install Docker

```bash
sudo yum install -y docker
sudo systemctl enable docker && sudo systemctl start docker
```

### Install kubeadm, kubelet & kubectl

```bash
sudo bash -c 'cat > /etc/yum.repos.d/kubernetes.repo' << EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
sudo setenforce 0
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet && sudo systemctl start kubelet
```

### Install kubernetes

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=stable-1.11
```

### Configurate kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

### Check

#### Check components

```bash
[centos@demo ~]$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}  
```

#### Check pods

```bash
[centos@demo ~]$ kubectl get po --all-namespaces
NAMESPACE     NAME                                                                    READY     STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-jdfsl                                                0/1       Pending   0          1m
kube-system   coredns-78fcdf6894-jrdpv                                                0/1       Pending   0          1m
kube-system   etcd-demo.eu-central-1.compute.internal                                 1/1       Running   0          1m
kube-system   kube-apiserver-demo.eu-central-1.compute.internal                       1/1       Running   0          51s
kube-system   kube-controller-manager-demo.eu-central-1.compute.internal              1/1       Running   0          1m
kube-system   kube-proxy-w8jdr                                                        1/1       Running   0          1m
kube-system   kube-scheduler-demo.eu-central-1.compute.internal                       1/1       Running   0          1m

```


### Install Canal

```bash
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/canal/rbac.yaml
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/canal/canal.yaml
```

#### Check pods

```bash
[centos@demo ~]$ kubectl get po --all-namespaces
NAMESPACE     NAME                                                                    READY     STATUS    RESTARTS   AGE
kube-system   canal-87jrb                                                             3/3       Running   0          6m
kube-system   coredns-78fcdf6894-jdfsl                                                1/1       Running   0          13m
kube-system   coredns-78fcdf6894-jrdpv                                                1/1       Running   1          13m
kube-system   etcd-demo.eu-central-1.compute.internal                                 1/1       Running   0          13m
kube-system   kube-apiserver-demo.eu-central-1.compute.internal                       1/1       Running   0          12m
kube-system   kube-controller-manager-demo.eu-central-1.compute.internal              1/1       Running   0          13m
kube-system   kube-proxy-w8jdr                                                        1/1       Running   0          13m
kube-system   kube-scheduler-demo.eu-central-1.compute.internal                       1/1       Running   0          13m

```

### Master Isolation

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```


# 2. Deploy WP

## Create PersistentVolume

```bash
kubectl create -f https://raw.githubusercontent.com/containerum/sk-workshop/master/wp/pv.yaml
```

```bash
[centos@demo ~]$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
mysql-pv   10Gi       RWO            Retain           Available             manual                   7s
wp-pv      10Gi       RWO            Retain           Available             manual                   7s


```


## Create PersistentVolumeClaim

```bash
kubectl create -f https://raw.githubusercontent.com/containerum/sk-workshop/master/wp/pvc.yaml
```

```bash
[centos@demo ~]$ kubectl get pvc
NAME        STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Bound     mysql-pv   10Gi       RWO            manual         7s
wp-pvc      Bound     wp-pv      10Gi       RWO            manual         7s

```

## Create Secret for MySQL Password

``` bash
kubectl create secret generic mysql-pass --from-literal=password=strongpassword
```

```bash
[centos@demo ~]$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-546hk   kubernetes.io/service-account-token   3         39m
mysql-pass            Opaque                                1         12s
```

## Deploy MySQL

```bash
kubectl create -f https://raw.githubusercontent.com/containerum/sk-workshop/master/wp/mysql.yaml
```


## Deploy WP

```bash
kubectl create -f https://raw.githubusercontent.com/containerum/sk-workshop/master/wp/wp.yaml
```

# 3. Deploy Containerum

## Deploy Helm

```bash
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

```bash
kubectl create -f https://raw.githubusercontent.com/containerum/sk-workshop/master/helm/rbac.yaml
helm init --service-account tiller
```

## Deploy Ingress Controller

```bash
sudo yum install git
git clone https://github.com/nginxinc/kubernetes-ingress.git
cd kubernetes-ingress/deployments/
```

```bash
kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f common/default-server-secret.yaml
kubectl apply -f rbac/rbac.yaml
kubectl apply -f deployment/nginx-ingress.yaml
```

Check:
```bash
kubectl get pods --namespace=nginx-ingress
```

## Deploy Containerum

```bash
helm repo add containerum https://charts.containerum.io
helm repo update
helm install containerum/containerum
```
