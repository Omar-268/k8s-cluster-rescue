# k8s-cluster-rescue
A collection of Kubernetes cluster scenarios with intentional misconfigurations, failures, and errors. 
Each scenario includes detailed steps to reproduce the issue and a guided troubleshooting process to fix it. 
Perfect for practicing real-world Kubernetes debugging skills.

## Labs Index

| Lab | Description | Link |
|-----|-------------|------|
| Lab 1 |  Kube Controller Manager misconfiguration causing `CrashLoopBackOff` | [Go to Lab 1](./Kube%20Controller%20Manager%20Misconfigured) |
| Lab 2 | Kubelet misconfiguration leading to `Node NotReady` | [Go to Lab 2](/Kubelet%20Misconfigured) |
| Lab 3 | Kubelet misconfiguration on **controlplane** causing `Node NotReady` | [Go to Lab 3](./Kubelet%20Misconfigured%20-%202) |
| Lab 4 | Pod pending after making some changes to the pod in the cluster | [Go to Lab 4](/Pod%20Issue%20-%201) |
| Lab 5 | Pod unschedulable due to **nodeAffinity mismatch** and taints | [Go to Lab 5](./Pod%20Issue%20-%202) |
| Lab 6 | Port-forward to nginx-service hangs, unable to access app | [Go to Lab 6](./Pod%20Issue%20-%203) |
| Lab 7 | Deployment fails due to misconfigured template | [Go to Lab 7](./Deployment-Issue) |
| Lab 8 | Deployment fails due to scheduling issue | [Go to Lab 8](./Deployment-Issue%20-%202) |

---

Each lab includes:
- Problem description  
- Step-by-step troubleshooting  
- Final solution  
