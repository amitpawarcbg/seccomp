# seccomp - **Restrict a Container's Syscalls with seccomp**

Seccomp stands for secure computing mode and has been a feature of the Linux kernel since version 2.6.12. It can be used to sandbox the privileges of a process, restricting the calls it is able to make from userspace into the kernel. Kubernetes lets you automatically apply seccomp profiles loaded onto a node to your Pods and containers.

Identifying the privileges required for your workloads can be difficult. In this tutorial, you will go through how to load seccomp profiles into a local Kubernetes cluster, how to apply them to a Pod, and how you can begin to craft profiles that give only the necessary privileges to your container processes.

Objectives
* Learn how to load seccomp profiles on a node
* Learn how to apply a seccomp profile to a container
* Observe auditing of syscalls made by a container process
* Observe behavior when a missing profile is specified
* Observe a violation of a seccomp profile
* Learn how to create fine-grained seccomp profiles
* Learn how to apply a container runtime default seccomp profile

In order to complete all steps in this tutorial, you must have kubernetes cluster ready or you can use [kind](https://kubernetes.io/docs/tasks/tools/#kind) and [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) to create kubernetes cluster.


**Note** : It is not possible to apply a seccomp profile to a container running with "privileged: true" set in the container's securityContext. Privileged containers always run as Unconfined.

# Let's go through some concepts.

1. **Linux Syscalls**

![Linux Syscalls](https://user-images.githubusercontent.com/88305831/177759943-aa9fcf00-50fa-4a44-b036-d4aaa43c86b9.png)

2. **Why we need SecComp**

![why we need SecComp](https://user-images.githubusercontent.com/88305831/177760892-c31a2248-f495-4882-8af9-ee7425240ecb.png)

3. **Sample SecComp Profile**

![image](https://user-images.githubusercontent.com/88305831/177761551-f8ed74f5-7285-4c13-b5c9-c77f618253d2.png)

4. **SecComp Profiles**

![image](https://user-images.githubusercontent.com/88305831/177762179-7ba7be1e-a1f2-499f-b442-a8c1fc420a70.png)

5. **SecComp Implementation**

![image](https://user-images.githubusercontent.com/88305831/177762821-24621e96-e816-4cca-96bf-9919cabbea9a.png)

6. **Apply to Pod**

![image](https://user-images.githubusercontent.com/88305831/177763515-b9a6638f-8049-4220-a2a1-2a65642f10b0.png)

![image](https://user-images.githubusercontent.com/88305831/177764070-3c1ae505-e8a2-4b3b-a431-371a1265550a.png)

---

# Before we begin please check below details :

1. Check seccomp is supported by kernel
* "grep -i seccomp /boot/config-$(uname -r)"

2. Default location for seccomp profiles on all nodes.
* "/var/lib/kubelet/seccomp"

If it is not present the create using below command.

* "mkdir -p /var/lib/kubelet/seccomp/profiles"

# Now let's start Implementing SecComp

1. **Download example seccomp profiles**

The contents of these profiles will be explored later on, but for now go ahead and download them into a directory named profiles/ so that they can be loaded into the cluster.

Run these commands:

$ "cd /var/lib/kubelet/seccomp"

$ "curl -L -o profiles/audit.json https://k8s.io/examples/pods/security/seccomp/profiles/audit.json"

$ "curl -L -o profiles/violation.json https://k8s.io/examples/pods/security/seccomp/profiles/violation.json"

$ "curl -L -o profiles/fine-grained.json https://k8s.io/examples/pods/security/seccomp/profiles/fine-grained.json"

$ "cd profiles; ls"

You should see three profiles listed at the end of the final step:

![image](https://user-images.githubusercontent.com/88305831/177765428-f10febc4-177d-46d7-a360-e35158c8bb3e.png)

2. **Create a Pod with a seccomp profile for syscall auditing**

To start off, apply the audit.json profile, which will log all syscalls of the process, to a new Pod.

$ "cat audit-pod.yaml" 

![image](https://user-images.githubusercontent.com/88305831/177770581-530410fb-3133-4f41-92be-155605dc0dc7.png)

Create the Pod in the cluster:

$ "kubectl apply -f audit-pod.yaml"
$ "kubectl get pod/audit-pod"

![image](https://user-images.githubusercontent.com/88305831/177771126-b141e23d-d315-4b43-9162-f7d41c0fef88.png)






