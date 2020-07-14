# Running Backup & Restore on an OpenShift (CRC) environment using Velero and Minio

The goal of this post is to provide a step-by-step tutorial on how to set up, backup and restore a WordPress application
running on CodeReady Containers OpenShift, using Velero for Backup and Restore and Minio as S3-like Object Storage.

From velero [documentation](https://velero.io/docs/v1.0.0/restic/) we find that velero allows the user to take snapshots of persistent volumes as part of the backups if you are using one of the supported cloud providers’ block storage offerings (Amazon EBS Volumes, Azure Managed Disks, Google Persistent Disks).

But, if you are using a volume type that does not have a native snapshot concept, velero integrates with restic to enable it. The downside is that restic does not support *hostPath* volume type, so to make this tutorial possible, we chose to use the *local* volume type.

This is part of a series of introductions to backup and restore tools I'm playing with. If you are interested also in this similar tests but using Kubernetes, check [Velero Blog Post on Kubernetes](https://tellesnobrega.github.io/velero-demo/) and if you are interested in Kanister, check [Kanister Blog Post on Kubernetes](https://tellesnobrega.github.io/kanister-demo/) and also the [Stash Blog Post on Kubernetes](https://tellesnobrega.github.io/stash-demo/).

## Setting up the Environment

We need to start by setting up the CRC Openshift environment.

### Install docker
```
sudo dnf install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Running minio container
```
docker pull minio/minio
docker run -p 9000:9000 --name minio -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=minio123" -v /mnt/data:/data minio/minio server /data
```

### Install CRC
```
sudo dnf install NetworkManager
crc setup
crc start
crc oc-env | head -1
 - Run the printed command
```

### Create loopback disks on the CRC machine to use as local volumes
```
oc get nodes
oc debug nodes/<node_name_from_previous_command>
cd /home
dd if=/dev/zero of=loopbackfile.img bs=100M count=10
dd if=/dev/zero of=loopbackfile2.img bs=100M count=10
du -sh loopbackfile.img
du -sh loopbackfile2.img
losetup -fP loopbackfile.img
losetup -fP loopbackfile2.img
mkfs.ext4 /home/loopbackfile.img
mkfs.ext4 /home/loopbackfile2.img 
mkdir /mnt/loopfs1
mkdir /mnt/loopfs2
mount -o loop /dev/loop0 /loopfs1
mount -o loop /dev/loop1 /loopfs2
df -hP /loopfs1/
df -hP /loopfs2/

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

### Deploy WordPress application

##### Clone this repo
```
git clone https://github.com/tellesnobrega/openshift-velero-demo.git
```

##### Deploy application
```
oc new-project wordpress
oc create -n wordpress secret generic mysql-pass --from-literal=password=<YOUR_PASSWORD>
oc apply -n wordpress -f openshift-velero-demo/mysql-deployment.yaml
oc apply -n wordpress -f openshift-velero-demo/wordpress-deployment.yaml
oc expose service/wordpress
```
##### Check for wordpress url
```
oc get routes
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
oc -n YOUR_POD_NAMESPACE annotate pod/YOUR_POD_NAME backup.velero.io/backup-volumes=YOUR_VOLUME_NAME_1,YOUR_VOLUME_NAME_2,...

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
oc delete project wordpress
```

##### Run restore command

To restore the application from backup you need to run the command below:

```
velero restore create --from-backup BACKUP_NAME OPTIONS...
```

And that is all I have for today!!!
