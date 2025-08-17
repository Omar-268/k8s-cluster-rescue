# Scenario: Pod Issue

## Problem Description
nginx-pod exposed to service nginx-service,
when port-forwarded kubectl port-forward svc/nginx-service 8080:80 it is stuck, so unable to access application curl http://localhost:8080

## How to Fix It

### Step 1 
The first thing we’ll do is verify the service itself.
```
controlplane:~$ kubectl describe service nginx-service 
Name:                     nginx-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=nginx-pod
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.104.94.130
IPs:                      10.104.94.130
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                                     # Empty, this is the problem
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```
We can notice that the endpoint field is empty.

### Step 2
Now let's check the pod 
```
controlplane:~$ kubectl describe pod nginx-pod 
Name:             nginx-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             node01/172.30.2.2
Start Time:       Sun, 17 Aug 2025 19:02:01 +0000
Labels:           <none>                    # no label
Annotations:      cni.projectcalico.org/containerID: 05e2fda9152e168bbf2cae559953f99608c00a86d905a3a635a1e05dc189d7a9
                  cni.projectcalico.org/podIP: 192.168.1.4/32
                  cni.projectcalico.org/podIPs: 192.168.1.4/32
Status:           Running
IP:               192.168.1.4
IPs:
  IP:  192.168.1.4
...
```
After inspecting the Pod, we noticed that it doesn’t have a label, even though the Service has a selector defined as `app=nginx-pod`.
So we’ll add this label (`app=nginx-pod`) to the Pod.
`kubectl label pod nginx-pod app=nginx-pod`

## Verification

Now, if we check the Service again, we’ll see that the endpoint field now contains an IP and port.
```
controlplane:~$ k describe service nginx-service 
Name:                     nginx-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=nginx-pod
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.104.94.130
IPs:                      10.104.94.130
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                192.168.1.4:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```
Let’s carry out the required step, which is exposing the nginx-pod through the nginx-service.
```
controlplane:~$ kubectl port-forward svc/nginx-service 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```
Let’s try running a curl command.
```
controlplane:~$ curl http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
Everything is working perfectly.

### Reference Lab
You can try this scenario yourself in the following lab: https://killercoda.com/sachin/course/CKA/pod-issue-8
