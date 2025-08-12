# Scenario: Kubelet Misconfigured

## 📝 Problem Description
Someone tried to improve the Kubelet on Node node01 , but broke it instead.

When you check the Nodes status, you will find that node01 is in NotReady state.
```
controlplane:~$ kubectl get nodes
NAME           STATUS     ROLES           AGE   VERSION
controlplane   Ready      control-plane   23d   v1.33.2
node01         NotReady   <none>          23d   v1.33.2
```
## 🛠️ How to Fix It

### Step 1 – SSH into Node01
Connect to the worker node (node01) via SSH to investigate the kubelet service, which is responsible for node–API server communication in Kubernetes. This step ensures we can directly access system logs, check service status, and troubleshoot potential startup issues on the node.
```
node01:~$ systemctl status kubelet.service 
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Tue 2025-08-12 12:54:31 UTC; 7s ago
       Docs: https://kubernetes.io/docs/
    Process: 6695 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM>
   Main PID: 6695 (code=exited, status=1/FAILURE)
        CPU: 36ms

Aug 12 12:54:31 node01 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Aug 12 12:54:31 node01 systemd[1]: kubelet.service: Failed with result 'exit-code'.
```
After checking the kubelet service, we noticed that it is in an auto-restart loop, failing with exit code 1.

### Step 2 – Check the Pod Logs
The next step is to check the kubelet logs to gather more detailed information and identify the exact cause of the failure. 
```
node01:~$ cat /var/log/syslog | grep kubelet
```
After reviewing the kubelet logs, it is clear that the service repeatedly fails to start due to an unrecognized flag(`--improve-speed`):
```
2025-08-12T13:00:30.196081+00:00 node01 kubelet[7513]: E0812 13:00:30.194826    7513 run.go:72] "command failed" err="failed to parse kubelet flag: unknown flag: --improve-speed"
2025-08-12T13:00:30.197449+00:00 node01 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
2025-08-12T13:00:30.197934+00:00 node01 systemd[1]: kubelet.service: Failed with result 'exit-code'.
```

### Fix the error
To fix this issue, we need to remove the unknown flag from the kubelet startup configuration.
Navigate to the file:
```
/var/lib/kubelet/kubeadm-flags.env
```
You will see an output like this: 
```
KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.10 --improve-speed"
```
Now remove the unrecognized flag `--improve-speed`

## ✅ Verification
After removing the invalid flag go back to the controlplane.
you can now see that the Node is Ready.
```
controlplane:~$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   23d   v1.33.2
node01         Ready    <none>          23d   v1.33.2
```

🧩 Kubernetes issue solved — everything is back on track!






