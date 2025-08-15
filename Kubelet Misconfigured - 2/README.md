# Scenario: Kubelet Misconfigured

## Problem Description
In controlplane node, something problem with kubelet configuration files

When you check the Nodes status, you will find that `controlplane` is in NotReady state.
```
controlplane:~$ kubectl get nodes
NAME           STATUS     ROLES           AGE   VERSION
controlplane   NotReady   control-plane   26d   v1.33.2
node01         Ready      <none>          26d   v1.33.2
```
## How to Fix It

### Step 1
First, we will check `kubelet service`. using `systemctl status kubelet`
```
controlplane:~$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Fri 2025-08-15 13:04:17 UTC; 4s ago
       Docs: https://kubernetes.io/docs/
    Process: 11215 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEAD>
   Main PID: 11215 (code=exited, status=1/FAILURE)
        CPU: 37ms

Aug 15 13:04:17 controlplane systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Aug 15 13:04:17 controlplane systemd[1]: kubelet.service: Failed with result 'exit-code'.
```
We observed that: 
- the kubelet is continuously restarting.
- The main PID exits immediately (status=1/FAILURE).
This indicates that there is a fundamental issue with the kubelet configuration.

### Step 2
Let's now check kubelet logs. using `sudo journalctl -u kubelet -f`
```
Aug 15 13:10:16 controlplane kubelet[12165]: E0815 13:10:16.339508   12165 run.go:72] "command failed" err="failed to construct kubelet dependencies: unable to load client CA file /etc/kubernetes/pki/CA.CERTIFICATE: open /etc/kubernetes/pki/CA.CERTIFICATE: no such file or directory"
```
The root cause has been identified: the kubelet is unable to read the CA file, which prevents it from successfully connecting to the API server.
Now let's check kubelet configuration files 
```yaml
controlplane:~$ cat /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/CA.CERTIFICATE   ## CA.CERTIFICATE doesn't exist the right path is /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerRuntimeEndpoint: ""
cpuManagerReconcilePeriod: 0s
crashLoopBackOff: {}
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMaximumGCAge: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
    text:
      infoBufferSize: "0"
  verbosity: 0
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

The kubelet needs to read the Client CA to communicate securely with the API server.
The file specified in config.yaml is /etc/kubernetes/pki/CA.CERTIFICATE.

However, when we run:
`ls /etc/kubernetes/pki/`
we find that the file actually exists as ca.crt, not CA.CERTIFICATE.

Therefore, there is a mismatch between the filename in config.yaml and the actual file.

### Step 3
Let's now ckeck `/etc/kubernetes/kubelet.conf`
```
controlplane:~$ cat /etc/kubernetes/kubelet.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJSmRjU2M2bk5PN1F3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRBM01Ua3hNekEwTlRoYUZ3MHpOVEEzTVRjeE16QTVOVGhhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUN4U1ZSMXZvU2JzU0NrT3AzK1ZqNURDRlYzS3ZlZ3hDdW56YUpMZGNGMDlod09WRERhQ3RPbWN1b2oKcUZjQWx3YkF1R3lseHZ4VWppdzBqWVRkVWY0b3hkTkVzZElxZlBEL281SXBUdEszR3dHdkdITDV6d1BQQVRnMwoyNy9WZmJjVVBoZlBkMWs1dGNwWFNkOHc2Ri9sZmRYYjRnOFZ0eXNHdUlaQlZxUUVsbE9iT05xT25vVkcyRWVUCitNaFhySnRjMUptKzlYQTQ5VGt3dURMUkhUclFGbXF1eERiYTZsbVA0Y1l5WXpzY2FGdGx0TWhVRGFTY0dPOEIKMWNhREZkRE5RMjN1UjFnNi9oekRzeCtoV0RrT3hTWm0zbDljamc3ZUYxZ204b1NFTGd4TXk4VG5QUFhaQ0JKYgpOUG56V2k4ZHo0SlB0dFlWcndXN2Y3ZHpBK0xKQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJRdjdpQTVrS2N4cmhQU3JCQ2NPcmh4K3JKVHBUQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQ0lLZGppMnJVZApHbktwRW5oMldET1dqbldSZGhTSTBPb1lYcFBWcHkvY1Y3d3ZrRHc3Z0NwKzFjSzJYR3BiaUxwcllSNHFQang4CjFGT0pzbnY1MSt1SWNkS3ZDaDNOeHZGYUYzdE9iK3crK2NjQnBOc3c3VWN0QXNPRTRGUHlhdjhMUnltOVpUTGIKSXFqTU5UOTFWYlBDellqZjVSM3h2WHg3YnVXdzhsTENWdnlIemhtVVgxWmtKeGQxdUpuMU8yMEhiWWc1WmFoagpiemRiQnBqVC9Rb0lmUUN5dkFnTGNkbVg1ampuTVQ0V0dYckFpZmR5N2cyd1FMOFlJRXp5QVZwUEg1UC8xSW5tCm8zcmNWZWNWdURIWGZZT0V2bk5tck12VXRSMFgyRGhIZXJrSjFMTW9DYWNFQkg1MUxoN2NiRjNmZjBYMGVZaEgKRTA3d2gvWDVCa2pLCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://172.30.1.2:64433333     # invalid port 
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:node:controlplane
  name: system:node:controlplane@kubernetes
current-context: system:node:controlplane@kubernetes
kind: Config
preferences: {}
users:
- name: system:node:controlplane
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```
By inspecting /etc/kubernetes/kubelet.conf, we were able to identify two issues.
- Invalid API server port → The server field shows :64433333 
- Same file path for certificate and key → Both client-certificate and client-key point to the same file (/var/lib/kubelet/pki/kubelet-client-current.pem), which is incorrect.
ٍSolution:
- Change the Port form 64433333 to the right one → 6443
- Point client-certificate to the public cert and client-key to the private key in kubelet.conf
  ```
  client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
  client-key: /var/lib/kubelet/pki/kubelet.key
  ```
## Verification
After updating All config files and then restart the Kubelet service, controlplane now is in `Ready` state
```
controlplane:~$ k get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   27d   v1.33.2
node01         Ready    <none>          26d   v1.33.2
```
### Reference Lab
You can try this scenario yourself in the following lab: https://killercoda.com/sachin/course/CKA/kubelet-issue

For more info check → https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/
