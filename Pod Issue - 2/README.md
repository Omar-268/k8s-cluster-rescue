# Scenario: Pod Issue

## Problem Description
After making some changes to the pod in the cluster, we discovered that a pod named `frontend` was not running.
```
controlplane:~$ k get pods
NAME       READY   STATUS    RESTARTS   AGE
frontend   0/1     Pending   0          99s
```
## How to Fix It

### Step 1 
First, we will investigate the pod using `kubectl describe pod redis-pod`, and check the Events section.
```
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  2m26s  default-scheduler  0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```
This shows a scheduling issue: no nodes are available due to node affinity/selector mismatch and untolerated taints.

### Step 2
Now we will check if there is any label on the node using `kubectl describe node node01 | grep Labels`
```
controlplane:~$ k describe nodes node01 | grep frontend  
Labels:             NodeName=frontendnodes
```
Now we will check the Pod to see if it has the correct node selector or affinity:
`kubectl get pods frontend -o yaml`
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: NodeName
            operator: In
            values:
            - frontend
```
We can see that the Pod has a nodeAffinity configured, which is the main reason it cannot be scheduled.

`Note`: According to the lab instructions, we are not allowed to modify the frontend Pod.

Since the Pod has a nodeAffinity configured, we also cannot modify the Pod. Therefore, the solution must focus on the node configuration.

We will update the label on the node from `frontendnodes` to `frontend`, so that it matches the Pod's nodeAffinity. This will allow the Pod to be scheduled successfully.

## Verification
Now, check the pod’s status and you will see that it is finally Running
```
controlplane:~$ k get pods 
NAME       READY   STATUS    RESTARTS   AGE
frontend   1/1     Running   0          26m
```
### Reference Lab
You can try this scenario yourself in the following lab: https://killercoda.com/sachin/course/CKA/pod-issue-3
