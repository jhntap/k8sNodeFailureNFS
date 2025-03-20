# Trident Force Detach Test for NFS Backend

This repository contains sample YAML files for a PersistentVolumeClaim (PVC) and a deployment that can be used to test the force detach feature of NetApp Trident during non-graceful Kubernetes node failures.

### Overview

Starting with Kubernetes v1.28, non-graceful node shutdown (NGNS) is enabled by default. This feature allows storage orchestrators like Trident to forcefully detach volumes from nodes that are not responding. This is particularly useful in scenarios where a node becomes unresponsive due to a crash, hardware failure, or network partitioning.

The force detach feature ensures that storage volumes can be quickly and safely detached from failed nodes and attached to other nodes, minimizing downtime and potential data corruption.

* Force detach is available for onatp-nas with Trident 25.02.0 and above

### Testing Environment
All samples have been tested with the following environment:

* Trident with Kubernetes Advanced v6.0 (Trident 24.02.0 & Kubernetes 1.29.4) <https://labondemand.netapp.com/node/878>

Please note that while these samples have been tested in a specific environment, they may need to be adjusted for use in other setups.

### Test Steps for Force Detach Feature in Trident NFS Backend

The following steps outline how to test the force detach feature of the Trident with iSCSI backend. This feature ensures that a Persistent Volume Claim (PVC) can be detached from a failed node and reattached to a healthy node, allowing the pod that uses the PVC to be rescheduled.

#### 1. Create the PVC
Create a Persistent Volume Claim (PVC) that will be used by the pod. 

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-nfs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: storage-class-nfs
```

Apply the PVC definition to the Kubernetes cluster:

```
# kubectl apply -f pvc-nfs.yaml
persistentvolumeclaim/pvc-nfs created
# kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATT
pvc-nfs   Bound    pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d   1Gi        RWO            storage-class-nfs   <unset>
```

#### 2. Create the deployment
Create a deployment that uses the PVC created in the previous step. 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  labels:
    app: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      volumes:
        - name: nfs-vol
          persistentVolumeClaim:
            claimName: pvc-nfs
      containers:
      - name: alpine
        image: alpine:3.19.1
        command:
          - /bin/sh
          - "-c"
          - "sleep 7d"
        volumeMounts:
          - mountPath: "/data"
            name: nfs-vol
      tolerations:
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
        tolerationSeconds: 30
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
        tolerationSeconds: 30
      nodeSelector:
        kubernetes.io/os: linux
```

Apply the deployment to the Kubernetes cluster:

```
# k apply -f test-deployment.yaml
deployment.apps/test-deployment created
# ./verify_status.sh
NAME              READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES          SELECTOR
test-deployment   1/1     1            1           6m3s   alpine       alpine:3.19.1   app=test
NAME                         DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES          SELECTOR
test-deployment-5ccb7f559f   1         1         1       6m3s   alpine       alpine:3.19.1   app=test,pod-template-hash=5ccb7f559f
NAME                               READY   STATUS    RESTARTS   AGE    IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-5ccb7f559f-vv8vw   1/1     Running   0          6m3s   192.168.28.108   rhel2   <none>           <none>
```

Identify the node running the pod from the deployment.

```
# kubectl get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-f64c69647-888g8   1/1     Running   0          104m   192.168.28.106   rhel2   <none>           <none>
```

Verify NFS PVC attachment to the node.

```
[root@rhel2 ~]# mount | grep 192.168.0.131
192.168.0.131:/trident_pvc_cf0f1a92_5c71_4708_97aa_b6973c3c907d on /var/lib/kubelet/pods/531cb297-2b06-43d2-9fc7-79ba7749eec1/volumes/kubernetes.io~csi/pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d/mount type nfs4 (rw,relatime,vers=4.2,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.0.62,local_lock=none,addr=192.168.0.131)
```

#### 3. Shutdown Node Running the Pod

```
[root@rhel2 ~]# shutdown -h now
```

Chech the status of the pod.

```
# after 30 seconds
# ./verify_status.sh
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment   0/1     1            0           11m   alpine       alpine:3.19.1   app=test
NAME                         DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment-5ccb7f559f   1         1         0       11m   alpine       alpine:3.19.1   app=test,pod-template-hash=5ccb7f559f
NAME                               READY   STATUS              RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-5ccb7f559f-6kscn   0/1     ContainerCreating   0          8s    <none>           rhel1   <none>           <none>
test-deployment-5ccb7f559f-vv8vw   1/1     Terminating         0          11m   192.168.28.108   rhel2   <none>           <none>

# k describe pod test-deployment-5ccb7f559f-vv8vw
Name:             test-deployment-5ccb7f559f-vv8vw
Namespace:        default
Priority:         0
Service Account:  default
Node:             rhel2/192.168.0.62
Start Time:       Thu, 20 Mar 2025 12:49:48 +0000
Labels:           app=test
                  pod-template-hash=5ccb7f559f
Annotations:      cni.projectcalico.org/containerID: 05f6c964b572563972e7d8e2be42b8d13816d9df4f93696a28adfea229f57968
                  cni.projectcalico.org/podIP: 192.168.28.108/32
                  cni.projectcalico.org/podIPs: 192.168.28.108/32
Status:           Running
IP:               192.168.28.108
IPs:
  IP:           192.168.28.108
Controlled By:  ReplicaSet/test-deployment-5ccb7f559f
Containers:
  alpine:
    Container ID:  cri-o://53d0da76d79660a3edeccd82905d8f53b74aa50064bf610227b601f79d305bc4
    Image:         alpine:3.19.1
    Image ID:      docker.io/library/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 7d
    State:          Running
      Started:      Thu, 20 Mar 2025 12:49:55 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from nfs-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rz48t (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       False
  ContainersReady             True
  PodScheduled                True
Volumes:
  nfs-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-nfs
    ReadOnly:   false
  kube-api-access-rz48t:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 30s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 30s
Events:
  Type     Reason                  Age   From                     Message
  ----     ------                  ----  ----                     -------
  Normal   Scheduled               10m   default-scheduler        Successfully assigned default/test-deployment-5ccb7f559f-vv8vw to rhel2
  Normal   SuccessfulAttachVolume  10m   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d"
  Normal   Pulled                  10m   kubelet                  Container image "alpine:3.19.1" already present on machine
  Normal   Created                 10m   kubelet                  Created container alpine
  Normal   Started                 10m   kubelet                  Started container alpine
  Warning  NodeNotReady            25s   node-controller          Node is not ready

# k describe pod test-deployment-5ccb7f559f-6kscn
Name:             test-deployment-5ccb7f559f-6kscn
Namespace:        default
Priority:         0
Service Account:  default
Node:             rhel1/192.168.0.61
Start Time:       Thu, 20 Mar 2025 13:00:45 +0000
Labels:           app=test
                  pod-template-hash=5ccb7f559f
Annotations:      <none>
Status:           Pending
IP:
IPs:              <none>
Controlled By:    ReplicaSet/test-deployment-5ccb7f559f
Containers:
  alpine:
    Container ID:
    Image:         alpine:3.19.1
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 7d
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from nfs-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lqr8d (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   False
  Initialized                 True
  Ready                       False
  ContainersReady             False
  PodScheduled                True
Volumes:
  nfs-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-nfs
    ReadOnly:   false
  kube-api-access-lqr8d:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 30s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 30s
Events:
  Type     Reason              Age   From                     Message
  ----     ------              ----  ----                     -------
  Normal   Scheduled           45s   default-scheduler        Successfully assigned default/test-deployment-5ccb7f559f-6kscn to rhel1
  Warning  FailedAttachVolume  45s   attachdetach-controller  Multi-Attach error for volume "pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d" Volume is already used by pod(s) test-deployment-5ccb7f559f-vv8vw
```

#### 4. Apply Taint to the failed Node

```
# kubectl taint nodes rhel2 node.kubernetes.io/out-of-service=nodeshutdown:NoExecute
node/rhel2 tainted
```

#### 5. Verify NFS PVC Detachment and Pod Rescheduling

Verify that the pod has been rescheduled onto another node and that it has successfully mounted the iSCSI PVC.

```
# ./verify_status.sh
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment   1/1     1            1           13m   alpine       alpine:3.19.1   app=test
NAME                         DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment-5ccb7f559f   1         1         1       13m   alpine       alpine:3.19.1   app=test,pod-template-hash=5ccb7f559f
NAME                               READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
test-deployment-5ccb7f559f-6kscn   1/1     Running   0          2m28s   192.168.26.5   rhel1   <none>           <none>

# k describe pod test-deployment-5ccb7f559f-6kscn
Name:             test-deployment-5ccb7f559f-6kscn
Namespace:        default
Priority:         0
Service Account:  default
Node:             rhel1/192.168.0.61
Start Time:       Thu, 20 Mar 2025 13:00:45 +0000
Labels:           app=test
                  pod-template-hash=5ccb7f559f
Annotations:      cni.projectcalico.org/containerID: 3389d06a34ccbaa2e1ee69f24b95e0145e02291c375008cf308e6fd9061662cb
                  cni.projectcalico.org/podIP: 192.168.26.5/32
                  cni.projectcalico.org/podIPs: 192.168.26.5/32
Status:           Running
IP:               192.168.26.5
IPs:
  IP:           192.168.26.5
Controlled By:  ReplicaSet/test-deployment-5ccb7f559f
Containers:
  alpine:
    Container ID:  cri-o://cc3a92c33cd8c72eca19525ec9996302cba275d088d31157740823c59f7b4eca
    Image:         alpine:3.19.1
    Image ID:      docker.io/library/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 7d
    State:          Running
      Started:      Thu, 20 Mar 2025 13:03:09 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from nfs-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lqr8d (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  nfs-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-nfs
    ReadOnly:   false
  kube-api-access-lqr8d:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 30s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 30s
Events:
  Type     Reason                  Age    From                     Message
  ----     ------                  ----   ----                     -------
  Normal   Scheduled               9m55s  default-scheduler        Successfully assigned default/test-deployment-5ccb7f559f-6kscn to rhel1
  Warning  FailedAttachVolume      9m55s  attachdetach-controller  Multi-Attach error for volume "pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d" Volume is already used by pod(s) test-deployment-5ccb7f559f-vv8vw
  Normal   SuccessfulAttachVolume  7m32s  attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d"
  Normal   Pulled                  7m31s  kubelet                  Container image "alpine:3.19.1" already present on machine
  Normal   Created                 7m31s  kubelet                  Created container alpine
  Normal   Started                 7m31s  kubelet                  Started container alpine
```

Verify NFS PVC attachment to the healthy node

```
[root@rhel1 ~]# mount | grep 192.168.0.131
192.168.0.131:/trident_pvc_cf0f1a92_5c71_4708_97aa_b6973c3c907d on /var/lib/kubelet/pods/17aa0f12-5a89-4e6b-97e4-93a858de75af/volumes/kubernetes.io~csi/pvc-cf0f1a92-5c71-4708-97aa-b6973c3c907d/mount type nfs4 (rw,relatime,vers=4.2,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.0.61,local_lock=none,addr=192.168.0.131)
```







