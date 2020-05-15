# velero-demo on Centos 7

## Setup Environment

### Install docker
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Running minio container
```
docker pull minio/minio
docker run -p 9000:9000 --name minio -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=minio123" -v /mnt/data:/data minio/minio server /data
```
### Install Kubectl
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```

### Install minikube
```
sudo yum -y install conntrack
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-1.9.2-0.x86_64.rpm
sudo rpm -ivh minikube-1.9.2-0.x86_64.rpm
minikube start --driver=none
```

### Download Velero 1.3.2 Release
```
wget https://github.com/vmware-tanzu/velero/releases/download/v1.3.2/velero-v1.3.2-linux-amd64.tar.gz
tar zxf velero-v1.3.2-linux-amd64.tar.gz
sudo mv velero-v1.3.2-linux-amd64/velero /usr/local/bin/
rm -rf velero*

```

### Clone this repo
```
git clone https://github.com/tellesnobrega/velero-demo.git
```

### Install Helm
```
yum -y install openssl
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Install local-volume-provider
```
git clone https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner.git
cd sig-storage-local-static-provisioner/
kubectl create -f deployment/kubernetes/example/default_example_storageclass.yaml
helm template ./helm/provisioner > deployment/kubernetes/provisioner_generated.yaml
kubectl create -f deployment/kubernetes/provisioner_generated.yaml
```

### Install Velero in the Kubernetes Cluster
```
velero install \
   --provider aws \
   --use-restic \
   --plugins velero/velero-plugin-for-aws:v1.0.0 \
   --bucket kubedemo \
   --secret-file ./velero-demo/minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<ip>:9000
```

### Deploy wordpress application
```
mkdir /mnt/disks
for vol in pv1 pv2; do
    mkdir /mnt/disks/$vol
    mount -t tmpfs $vol /mnt/disks/$vol
done

kubectl create ns wordpress
kubectl create -n wordpress secret generic mysql-pass --from-literal=password=<YOUR_PASSWORD>
kubectl create -n wordpress -f velero-demo/mysql-deployment.yaml
kubectl create -n wordpress -f velero-demo/wordpress-deployment.yaml
```
#### Check for wordpress url
```
minikube -n wordpress service wordpress --url
