# Backup and restore etcd cluster on Kubernetes


## What this does?
In Unix like operating systems, ETC folder is used to keep configuration data. Similarly, Kubernetes uses ETCD for keeping configuration data as well as cluster information. The name ETCD comes from the ETC folder with an addition of the letter D which stands for distributed systems.

ETCD is the key value data store of the Kubernetes Cluster. The data is stored for service discovery and cluster management. It is important to take a backup of the ETCD as a measure against failures.

In order to interact with etcd, in terms of backup and restore purposes, we will utilize a command line tool: etcdctl

etcdctl has a snapshot option which makes it relatively easy to take a backup of the cluster. In the next section, I will show you how to back up an etcd cluster by using the etcdctl snapshot option.

### References
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/


## Backing up an etcd cluster
```
etcdctl version

```

If etcdctl is installed, you will see an output like the one above, if not it will give a `command not found `error.

### Install ETCDCTL & ETCDUTL on Master Node or Admin Node
```
sudo apt  install etcd-client -y

sudo wget https://github.com/etcd-io/etcd/releases/download/v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz
sudo tar xvf etcd-v3.5.16-linux-amd64.tar.gz\

cd etcd-v3.5.16-linux-amd64/
sudo cp etcdutl /usr/local/bin
```

First, we need information regarding the endpoints. If we have the etcd running in the same server, then we can simply add

```
ls -al /etc/kubernetes/manifests/
sudo cat /etc/kubernetes/manifests/etcd.yaml

```


### Backup ETCD Command

```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>

```

How do we get the endpoint and the certificate information?

Well, we can retrieve them from the etcd pod and the manifest file for etcd pod is located under /etc/kubernetes/manifests folder

```
sudo cat /etc/kubernetes/manifests/etcd.yaml

--endpoints > --listen-client-urls=https://127.0.0.1:2379
--cacert    > trusted-ca-file
--cert      > cert-file
--key       > key-file

```

127.0.0.1 being the localhost ip and 2379 being the official port number of etcd. If the etcd is running on another server, then we need to change the localhost ip with the ip of that server.

Secondly, we need certificates to authenticate to the ETCD server to take the backup. The certificates that are required are

```
cat /etc/kubernetes/manifests/etcd.yaml | grep listen
cat /etc/kubernetes/manifests/etcd.yaml | grep file

```

Once we have the necessary information, we can run the snapshot save command using etcdctl.

One thing to note here is that we need to place ETCDCTL_API=3 at the beginning of the command. API version used by etcdctl to speak to etcd may be set to version 2 or 3 via the ETCDCTL_API environment variable. However, we need to make sure it is default to the v3 API in order to take a snapshot.


```
ETCDCTL_API=3 etcdctl snapshot
export ETCDCTL_API=3

```

```
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt  \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save <backup-file-location>

  sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt  \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db

```

```
ls -al /opt/etcd-backup.db

du -sh /opt/etcd-backup.db

sudo etcdctl --write-out=table snapshot status /opt/etcd-backup.db

```

We now have a backup of the etcd!

## Restoring an etcd cluster

Caution:
If any API servers are running in your cluster, you should not attempt to restore instances of etcd. Instead, follow these steps to restore etcd:

stop all API server instances
restore state in all etcd instances
restart all API server instances

etcd supports restoring from snapshots that are taken from an etcd process of the major.minor version. Restoring a version from a different patch version of etcd is also supported. A restore operation is employed to recover the data of a failed cluster.

```

export ETCDCTL_API=3
etcdctl --data-dir <data-dir-location> snapshot restore snapshot.db

sudo cat /etc/kubernetes/manifests/etcd.yaml

 --data-dir=/var/lib/etcd

```

Note : we are creating etcd-new new directory on location /var/lib/

we can move on to actually restoring etcd. Imagine that etcd somehow failed and we need to revert it to the last saved state. We know that we have an etcd-backup.db which we saved earlier.

```

sudo etcdctl --data-dir /var/lib/etcd-new  snapshot restore /opt/etcd-backup.db

or 

sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt  \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --data-dir /var/lib/etcd-new \
   snapshot restore /opt/etcd-backup.db

```

```

ls -al /var/lib/ | grep etcd
sudo ls -al /var/lib/etcd-new/

```

As you know, kubernetes keep manifest of static pods in the /etc/kubernetes/manifests folder. etcd.yaml, which is the manifest file for the etcd pod is also there. We need to edit that file and make sure that it uses the new restored data directory as volume instead of the old one.

```

cd /etc/kubernetes/manifests/

sudo mkdir my-dir

sudo mv *.yaml  my-dir/

cd my-dir

sudo vim etcd.yaml

```

Change the following parts in the etcd.yaml

- 1- Change

```
--data-dir=/var/lib/etcd        change to     --data-dir=/var/lib/etcd-new

```
- 2- Change

```
volumeMounts:
    - mountPath: /var/lib/etcd

change to  


 volumeMounts:
    - mountPath: /var/lib/etcd-new

```

- 3- Change

```

 - hostPath:
      path: /var/lib/etcd


change to  

- hostPath:
      path: /var/lib/etcd-new

```

At this point, you will need to wait couple minutes for the etcd pod to start with the new state. Meanwhile you will not be able to get response from the apiserver.

```

sudo mv * ./..

ls -al my-dir/

sudo rm -rf my-dir/

k get pods -n kube-system

k describe pods etcd-k8s-master -n kube-system

```


```

sudo systemctl status kubelet 

sudo systemctl restart kubelet 
sudo systemctl daemon-reload
sudo systemctl restart kubelet 

sudo systemctl status kubelet 

```







































