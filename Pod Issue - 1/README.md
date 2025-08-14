# Scenario: Pod Issue

## Problem Description
After making some changes to the pod in the cluster, we discovered that a pod named `redis-pod` was not running.
```
controlplane:~$ k get pods
NAME        READY   STATUS    RESTARTS   AGE
redis-pod   0/1     Pending   0          5m34s
```

## How to Fix It

### Step 1 
First, we will investigate the pod using `kubectl describe pod redis-pod`.
```
controlplane:~$ kubectl describe pod redis-pod
Name:             redis-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none>
Labels:           <none>
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Containers:
  redis-container:
    Image:        redis:latested     #⚠️⚠️ Invalid image tag
    Port:         6379/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:
      /data from redis-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-p6xgl (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  redis-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-redis       ## ⚠️⚠️ wrong ClaimName
    ReadOnly:   false
  kube-api-access-p6xgl:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  10m    default-scheduler  0/2 nodes are available: persistentvolumeclaim "pvc-redis" not found. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
  Warning  FailedScheduling  4m42s  default-scheduler  0/2 nodes are available: persistentvolumeclaim "pvc-redis" not found. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```
- The output shows that the image tag is invalid and should be changed to `redis:latest`.
- The Events section also indicates that the PersistentVolumeClaim pvc-redis was not found.

### Step 2
Following the previous step, we will now inspect the PersistentVolumeClaim (PVC).
using the command `kubectl get pvc`
```
controlplane:~$ kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
redis-pvc   Pending                                      manually       <unset>                 19m
```
Now the root cause has been identified, the pod is configured to use a PVC named `pvc-redis`, but kubectl get pvc shows that the actual PVC name is `redis-pvc` . This name mismatch prevents the pod from mounting the volume.
Let’s go ahead and fix this in the pod configuration

### Step 3
When checking the pod now, we can see that it is in a Pending state.
```
controlplane:~$ k get pods
NAME        READY   STATUS    RESTARTS   AGE
redis-pod   0/1     Pending   0          2m26s
```
So What is the problem? 
Let’s check the pod again and take a look at the Events section.
` kubectl describe pod redis-pod `
```
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  4m15s  default-scheduler  0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```
There is still a problem related to the PersistentVolumeClaim (PVC) that needs to be resolved.
Let’s go and inspect the PV and PVC

```
controlplane:~$ k edit pv 
persistentvolume/redis-pv edited
controlplane:~$ k describe pvc
Name:          redis-pvc
Namespace:     default
StorageClass:  manually
Status:        Bound
Volume:        redis-pv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      100Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       redis-pod
Events:
  Type     Reason              Age                    From                         Message
  ----     ------              ----                   ----                         -------
  Warning  ProvisioningFailed  3m57s (x162 over 44m)  persistentvolume-controller  storageclass.storage.k8s.io "manually" not found
```
Clearly, the problem is that the StorageClass `manually` does not exist. 
Let’s go and inspect the PV itself.
```
controlplane:~$ k describe pv     
Name:            redis-pv
Labels:          <none>
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    manual  ## ⚠️⚠️ mismatch with PVC
Status:          Available
Claim:           
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        100Mi
Node Affinity:   <none>
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /mnt/data/redis
    HostPathType:  
Events:            <none>
```
Update the StorageClass and set it to manual

## Verification
Now, check the pod’s status and you will see that it is finally Running
```
controlplane:~$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
redis-pod   1/1     Running   0          20m
```

### Reference Lab
You can try this scenario yourself in the following lab: https://killercoda.com/sachin/course/CKA/pod-issue-2

