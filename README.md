# velero-demo on Centos 7

### Setup Environment

Start your environment setup by following this [tutorial](https://github.com/tellesnobrega/velero-demo/blob/master/setup-environment.md)

### Add some content to WordPress

Now that the environment is set up you can add a post to WordPress

### Run backup

#### Annotate each pod that you want to backup with
```
kubectl -n YOUR_POD_NAMESPACE annotate pod/YOUR_POD_NAME backup.velero.io/backup-volumes=YOUR_VOLUME_NAME_1,YOUR_VOLUME_NAME_2,...

```
where YOUR_VOLUME_NAME_1, ... is the name of the volume found in the mysql-deploymet.yaml and wordpress-deployment.yaml for the respective pods.
```
volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
```

#### Run backup command
```
velero backup create NAME OPTIONS
```

#### Destroy WordPress deployment
```
kubectl delete ns wordpress
```

#### Run restore command
```
velero restore create --from-backup BACKUP_NAME OPTIONS...
kubectl -n wordpress patch svc wordpress -p '{"spec": { "type": "NodePort", "ports": [ { "nodePort": <PORT>, "port": 80, "protocol": "TCP", "targetPort": 80 } ] } }'

```

Replace <PORT> with the port from previous wordpress deployment. This is needed because wordpress keeps url information on the database and after the restore minikube gives the service a new PORT. Patching this solves this redirecting issue.
