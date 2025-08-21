# Scenario: Deployment-Issue

## Problem Description
database-deployment deployment pods are not running, when checking deployment status we notice it's not running.
```
controlplane:~$ k get deployments.apps 
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
database-deployment   0/1     1            0           3m15s
```
## How to Fix It

### Step 1 
Let's inspect the pod created by the deployment:
```
controlplane:~$ k get pods 
NAME                                  READY   STATUS    RESTARTS   AGE
database-deployment-b95f67975-xncgn   0/1     Pending   0          12m
```
We will find the pod in Pending state.
```
controlplane:~$ k describe pod database-deployment-b95f67975-xncgn
...
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  4m (x3 over 14m)  default-scheduler  0/2 nodes are available: persistentvolumeclaim "postgres-db-pvc" not found. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```
 After inspecting the pod, we noticed an issue in the events:
 - the pod could not be scheduled because the required PersistentVolumeClaim (postgres-db-pvc) does not exist.

### Step 2
Now, let's inspect the existing PVCs to see the problem:
```
controlplane:~$ k get pvc
NAME           STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-pvc   Pending   postgres-pv   0                         standard       <unset>                 18m
```
We noticed that there is no PVC named `postgres-db-pvc`.  
The correct PVC name is actually `postgres-pvc`.

Since the PVC name was wrong, we need to update it in the deployment manifest:
```yaml
kubectl edit deployment database-deployment
...
spec:
      containers:
      - env:
        - name: POSTGRES_DB
          value: mydatabase
        - name: POSTGRES_USER
          value: myuser
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              key: POSTGRES_PASSWORD
              name: postgres-secret
        image: postgres:latest
        imagePullPolicy: Always
        name: postgres-container
        ports:
        - containerPort: 5432
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-db-pvc   ### replace this with postgres-pvc
...
```
### Step 3
After updating the PVC name, check the deployment again:
```
controlplane:~$ k get deployments.apps 
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
database-deployment   0/1     1            0           26m
```
This means there are still issues preventing the pod from running, so we need to investigate further.

let's inspect the PersistentVolumeClaim (PVC):
```
Events:
  Type     Reason          Age                    From                         Message
  ----     ------          ----                   ----                         -------
  Warning  VolumeMismatch  3m34s (x102 over 28m)  persistentvolume-controller  Cannot bind to requested volume "postgres-pv": requested PV is too small
```
rom inspecting the  PVC, we noticed a **volume mismatch**:  
The PVC request does not match the PV specification, so the claim cannot be bound.

Now, let’s take a closer look at the PersistentVolume:
```
controlplane:~$ k describe pv
Name:            postgres-pv
Labels:          <none>
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Available
Claim:           
Reclaim Policy:  Retain
Access Modes:    RWO       ## should be the same in PVC manifest
VolumeMode:      Filesystem
Capacity:        100Mi    ## increase it to at least 150Mi
Node Affinity:   <none>
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /data/postgres
    HostPathType:  
Events:            <none>
```
From the PV details, we found two issues:

1. **Capacity mismatch** → The PVC requests more storage than what the PV provides.  
2. **Access mode mismatch** →  
   - PV is using `ReadWriteOnce (RWO)`  
   - PVC is requesting `ReadWriteMany (RWX)`
  
### Step 4

To solve the issue, make sure the **capacity** and **access modes** match between PV and PVC.  

-  the PV has `ReadWriteOnce`, the PVC must also request `ReadWriteOnce`.
-  Adjust the storage size so the PVC request ≤ PV capacity.

## Verification

Finally, check the deployment again:
```
controlplane:~$ k get deployments.apps 
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
database-deployment   1/1     1            1           37m
```

Now the deployment is Ready and all pods are running successfully

### Reference Lab
You can try this scenario yourself in the following lab:https://killercoda.com/sachin/course/CKA/deployment-issue-4
  
  
