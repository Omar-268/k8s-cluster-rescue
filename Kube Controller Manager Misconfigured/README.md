# Scenario: Kube Controller Manager Misconfigured

## 📝 Problem Description
A custom Kube Controller Manager container image was running in this cluster for testing. It has been reverted back to the default one, but it's not coming back up. Fix it.

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
