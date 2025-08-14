# Scenario: Pod Issue

## Problem Description
`postgres-deployment.yaml` template is there, now we can't create object due to some issue in that.

## How to Fix It

### Step 1 
First, let's take a look at `postgres-deployment.yaml`
```yaml
controlplane:~$ cat postgres-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres-container
          image: postgres:latest
          env:
            - name: POSTGRES_DB
              value: mydatabase
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrte   ## might be wrong
                  key: db_user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db_password
          ports:
            - containerPort: 5432
```
We can see there is mismatch between `name.secretKeyRef` in both POSTGRES_USER and POSTGRES_PASSWORD.
To confirm, I will check the secret using `kubectl get secrets`.
```
controlplane:~$ kubectl get secrets 
NAME              TYPE     DATA   AGE
postgres-secret   Opaque   2      8m15s
```
this indicates that name.secretKeyRef at POSTGRES_USER should be updated to postgres-secret

### Step 2
Next, we apply the Deployment and check if it runs successfully.
```
controlplane:~$ k apply -f postgres-deployment.yaml 
deployment.apps/postgres-deployment created
controlplane:~$ k get pods
NAME                                   READY   STATUS                       RESTARTS   AGE
postgres-deployment-654b5ddb45-5vmd5   0/1     CreateContainerConfigError   0          83s
```
Once the Deployment was applied, the Pod failed to start with the error CreateContainerConfigError.
Let's check and find out where the issue is coming from. using `kubectl describe pod <pod name>`,and look at Events section 
```
Warning  Failed       5s (x17 over 3m26s)    kubelet            Error: couldn't find key db_user in Secret default/postgres-secret
```
We can see that the Pod failed because the `db_user` key is missing in the `postgres-secret`.
Let's check the postgres-secret to see where the problem is.
```
controlplane:~$ k get secrets postgres-secret -o yaml 
apiVersion: v1
data:
  password: ZGJwYXNzd29yZAo=
  username: ZGJ1c2VyCg==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"ZGJwYXNzd29yZAo=","username":"ZGJ1c2VyCg=="},"kind":"Secret","metadata":{"annotations":{},"name":"postgres-secret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2025-08-14T18:40:43Z"
  name: postgres-secret
  namespace: default
  resourceVersion: "4505"
  uid: 3ad53681-768c-4e6e-84b1-b04efd18277f
type: Opaque
```
We can see the issue was a mismatch between the postgres-secret and deployment file’s environment variables.
TO solve this, Update `POSTGRES_USER` and `POSTGRES_PASSWORD` to use the correct secret (postgres-secret) and keys (username and password), then reapply the deployment.

## Verification
Now, check the pod’s status and you will see that it is finally Running
```
controlplane:~$ k apply -f postgres-deployment.yaml 
deployment.apps/postgres-deployment configured
controlplane:~$ k get pods
NAME                                   READY   STATUS    RESTARTS   AGE
postgres-deployment-7db8b9499d-g82zb   1/1     Running   0          8s
```

### Reference Lab
You can try this scenario yourself in the following lab: https://killercoda.com/sachin/course/CKA/deployment-issue

