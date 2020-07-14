# Running Backup & Restore on a Kubernetes environment using Velero and Minio

The goal of this post is to provide a step-by-step tutorial on how to set up, backup and restore a WordPress application
running on Minikube, using Velero for Backup and Restore and Minio as S3-like Object Storage.

From velero [documentation](https://velero.io/docs/v1.0.0/restic/) we find that velero allows the user to take snapshots of persistent volumes as part of the backups if you are using one of the supported cloud providers’ block storage offerings (Amazon EBS Volumes, Azure Managed Disks, Google Persistent Disks).

But, if you are using a volume type that does not have a native snapshot concept, velero integrates with restic to enable it. The downside is that restic does not support *hostPath* volume type, so to make this tutorial possible, we chose to use the *local* volume type.

## Setting up the Environment

We need to start by setting up the kubernetes envrionment.

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

### Install Velero

Now we need to install velero to our kubernetes cluster.

##### Download Velero 1.3.2 Release
```
wget https://github.com/vmware-tanzu/velero/releases/download/v1.3.2/velero-v1.3.2-linux-amd64.tar.gz
tar zxf velero-v1.3.2-linux-amd64.tar.gz
sudo mv velero-v1.3.2-linux-amd64/velero /usr/local/bin/
rm -rf velero*
```
##### Install Velero in the Kubernetes Cluster
```
velero install \
   --provider aws \
   --use-restic \
   --plugins velero/velero-plugin-for-aws:v1.0.0 \
   --bucket kubedemo \
   --secret-file ./velero-demo/minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<ip>:9000
```

### Install local-volume-provider

In order to easy our management of the *local* volume types, we install the local-volume-provider. It detects the volumes under the chosen path and creates the persistent volumes to be used by the application.

##### Install Helm
```
yum -y install openssl
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

##### Install local-volume-provider
```
git clone https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner.git
cd sig-storage-local-static-provisioner/
kubectl create -f deployment/kubernetes/example/default_example_storageclass.yaml
helm template ./helm/provisioner > deployment/kubernetes/provisioner_generated.yaml
kubectl create -f deployment/kubernetes/provisioner_generated.yaml
```

### Deploy WordPress application

##### Clone this repo
```
git clone https://github.com/tellesnobrega/velero-demo.git
```

##### Deploy application
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
##### Check for wordpress url
```
minikube -n wordpress service wordpress --url
```

## Test Backup & Restore procedure

Now that the environment is set up you start the setting up the procedure by adding a post to WordPress

### Add some content to WordPress

Go to New Post -> Write a new post.

### Run backup

##### Annotate each pod that you want to backup

Each pod with a persistent volume needs to be annotated in order for restic to detect them and back them up.
For that run the command below:

```
kubectl -n YOUR_POD_NAMESPACE annotate pod/YOUR_POD_NAME backup.velero.io/backup-volumes=YOUR_VOLUME_NAME_1,YOUR_VOLUME_NAME_2,...

```
where YOUR_VOLUME_NAME_1, ... is the name of the volume found in the mysql-deploymet.yaml and wordpress-deployment.yaml for the respective pods.
i.e.:
```
volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
```

##### Run backup command
```
velero backup create NAME OPTIONS
```

##### Destroy WordPress deployment
```
kubectl delete ns wordpress
```

##### Run restore command

To restore the application from backup you need to run the command below:

```
velero restore create --from-backup BACKUP_NAME OPTIONS...
```

Side note, since WordPress stores its URL on the database. Once the service is provisioned again during restore, Kubernetes will attach a new port to the service and the URL given by *minikube -n wordpress service wordpress --url* won't really work because it will get redirect to the previous port.

To fix this, you need to patch the service with the correct port using the command below:

```
kubectl -n wordpress patch svc wordpress -p '{"spec": {"type":"NodePort","ports": [{"nodePort": <PORT>, "port": 80, "protocol": "TCP", "targetPort": 80 } ] } }'

```
Replace <PORT> with the port from previous wordpress deployment. This is needed because wordpress keeps url information on the database and after the restore minikube gives the service a new PORT. Patching this solves this redirecting issue.
        

And that is all I have for today!!!
