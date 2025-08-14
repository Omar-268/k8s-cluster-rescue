# Scenario: Kube Controller Manager Misconfigured

## Problem Description
A custom Kube Controller Manager container image was running in this cluster for testing. It has been reverted back to the default one, but it's not coming back up. 

When you check the pods, you will find that the kube-controller-manager pod is in CrashLoopBackOff.

```
controlplane:~$ k get pod -A
NAMESPACE            NAME                                      READY   STATUS             RESTARTS        AGE
kube-system          calico-kube-controllers-fdf5f5495-g6nd4   1/1     Running            1 (28m ago)     23d
kube-system          canal-grvvt                               2/2     Running            2 (28m ago)     23d
kube-system          coredns-6ff97d97f9-8v98s                  1/1     Running            1 (28m ago)     23d
kube-system          coredns-6ff97d97f9-9j8cr                  1/1     Running            1 (28m ago)     23d
kube-system          etcd-controlplane                         1/1     Running            1 (28m ago)     23d
kube-system          kube-apiserver-controlplane               1/1     Running            1 (28m ago)     23d
kube-system          kube-controller-manager-controlplane      0/1     CrashLoopBackOff   7 (2m54s ago)   14m
kube-system          kube-proxy-k9t79                          1/1     Running            1 (28m ago)     23d
kube-system          kube-scheduler-controlplane               1/1     Running            1 (28m ago)     23d
local-path-storage   local-path-provisioner-5c94487ccb-xxglc   1/1     Running            1 (28m ago)     23d
```
## How to Fix It

### Step 1 – Check the Pod Logs
The very first step in diagnosing this issue is to review the logs of the affected pod to identify any error messages or unusual behavior.
```
kubectl logs kube-controller-manager-controlplane -n kube-system
```
After running the above command, you will see the following message:
```
Error: unknown flag: --project-sidecar-insertion
```
### Step 2 – Inspect the Manifests
This error leads us to check the Kubernetes manifests to identify any incorrect or unsupported flags in the configuration.
Therefore, we will open the manifest file located at:
```
/etc/kubernetes/manifests/kube-controller-manager.yaml
```
When you inspect the manifest, you will indeed find the unusual flag that is causing the issue and is not recognized by Kubernetes.
Therefore, we will remove it.
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=192.168.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --project-sidecar-insertion=true  ## ⚠️⚠️ Remove this 
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --use-service-account-credentials=true
```
## Verification
After removing the invalid flag and re-checking the pod status, it should now be in the Running state.
```
NAMESPACE            NAME                                      READY   STATUS    RESTARTS      AGE
kube-system          calico-kube-controllers-fdf5f5495-g6nd4   1/1     Running   1 (53m ago)   23d
kube-system          canal-grvvt                               2/2     Running   2 (53m ago)   23d
kube-system          coredns-6ff97d97f9-8v98s                  1/1     Running   1 (53m ago)   23d
kube-system          coredns-6ff97d97f9-9j8cr                  1/1     Running   1 (53m ago)   23d
kube-system          etcd-controlplane                         1/1     Running   1 (53m ago)   23d
kube-system          kube-apiserver-controlplane               1/1     Running   1 (53m ago)   23d
kube-system          kube-controller-manager-controlplane      1/1     Running   0             3m36s
kube-system          kube-proxy-k9t79                          1/1     Running   1 (53m ago)   23d
kube-system          kube-scheduler-controlplane               1/1     Running   1 (53m ago)   23d
local-path-storage   local-path-provisioner-5c94487ccb-xxglc   1/1     Running   1 (53m ago)   23d
```

🧩 Kubernetes issue solved — everything is back on track!

### Reference Lab
You can try this scenario yourself in the following lab: https://killercoda.com/killer-shell-cka/scenario/kube-controller-manager-misconfigured
