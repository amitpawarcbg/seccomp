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

$ "kubectl get pod/audit-pod -o wide"

![image](https://user-images.githubusercontent.com/88305831/177772515-ffe5cc04-269e-44f3-9b5c-73414dddbbcc.png)


In order to be able to interact with this endpoint exposed by this container, create a NodePort Services that allows access to the endpoint from inside the kind control plane container.

$ "kubectl expose pod audit-pod --type NodePort --port 5678"

![image](https://user-images.githubusercontent.com/88305831/177771607-f647e60c-c7bf-458b-baf6-ca1e29bae77c.png)

Check what port the Service has been assigned on the node.

$ "kubectl get service audit-pod"

![image](https://user-images.githubusercontent.com/88305831/177771676-bd156e32-eaa2-4703-b1d6-76c948054873.png)

Now you can use curl to access that endpoint

![image](https://user-images.githubusercontent.com/88305831/177772063-4444c537-96db-470e-b2da-8d3a2d7bf983.png)

You can see that the process is running, but what syscalls did it actually make? Because this Pod is running in a local cluster on node "node1", you should be able to see those in /var/log/syslog. 

Open up a new terminal window for node1 and tail the output for calls from http-echo:

$ "tail -f /var/log/syslog | grep 'http-echo'"

You should already see some logs of syscalls made by http-echo, and if you curl the endpoint in the control plane container you will see more written.

For example:

![image](https://user-images.githubusercontent.com/88305831/177772730-b423d54a-c511-4f18-9b1d-9d97dfee415a.png)


You can begin to understand the syscalls required by the http-echo process by looking at the syscall= entry on each line. While these are unlikely to encompass all syscalls it uses, it can serve as a basis for a seccomp profile for this container.

Clean up that Pod and Service before moving to the next section:

$ "kubectl delete service audit-pod --wait"

$ "kubectl delete pod audit-pod --wait --now"

![image](https://user-images.githubusercontent.com/88305831/177773436-5404d8c9-e8b5-48e0-a7dd-4ff418713759.png)

---

**Create Pod with seccomp profile that causes violation**

For demonstration, apply a profile to the Pod that does not allow for any syscalls.

![image](https://user-images.githubusercontent.com/88305831/177984764-70ad0278-2eeb-4e0f-9f58-35c4304d7b6b.png)

$ "kubectl apply -f violation-pod.yaml"

The Pod creates, but there is an issue. If you check the status of the Pod, you should see that it failed to start.

$ "kubectl get pods -o wide"

![image](https://user-images.githubusercontent.com/88305831/177985120-c824dd48-af1a-4d09-8d3e-fbfd11615bb9.png)

As seen in the previous example, the http-echo process requires quite a few syscalls. Here seccomp has been instructed to error on any syscall by setting "defaultAction": "SCMP_ACT_ERRNO". This is extremely secure, but removes the ability to do anything meaningful. What you really want is to give workloads only the privileges they need.

Clean up that Pod before moving to the next section:

$ "kubectl delete pod violation-pod --wait --now"

---

**Create Pod with seccomp profile that only allows necessary syscalls**

If you take a look at the fine-grained.json profile, you will notice some of the syscalls seen in syslog of the first example where the profile set "defaultAction": "SCMP_ACT_LOG". Now the profile is setting "defaultAction": "SCMP_ACT_ERRNO", but explicitly allowing a set of syscalls in the "action": "SCMP_ACT_ALLOW" block. Ideally, the container will run successfully and you will see no messages sent to syslog.

![image](https://user-images.githubusercontent.com/88305831/177985486-62bba71b-4bf4-4811-968c-9c1b9e9db4a4.png)


Create the Pod in your cluster:

$ "kubectl apply -f fine-pod.yaml"

The Pod should be showing as having started successfully:

$ "kubectl get pod fine-pod"

![image](https://user-images.githubusercontent.com/88305831/177985661-eae1ab48-79a2-4e71-9dfa-8290698d23ed.png)



Open up a new terminal window and use tail to monitor for log entries that mention calls from http-echo:

$ "tail -f /var/log/syslog | grep 'http-echo'"

Next, expose the Pod with a NodePort Service:

$ "kubectl expose pod fine-pod --type NodePort --port 5678"

![image](https://user-images.githubusercontent.com/88305831/177988052-cfaf7a5c-e2d4-4b0b-ae37-0253236e06d3.png)


Check what port the Service has been assigned on the node:

$ "kubectl get service fine-pod"

![image](https://user-images.githubusercontent.com/88305831/177988094-25ae16d6-ce53-4fb4-80a1-1cb8d682a027.png)

Use curl to access that endpoint from inside the kind control plane container:

![image](https://user-images.githubusercontent.com/88305831/177988178-f71c42e5-0208-4d04-8ea7-33c30bd06e60.png)

You should see no output in the syslog. This is because the profile allowed all necessary syscalls and specified that an error should occur if one outside of the list is invoked. This is an ideal situation from a security perspective, but required some effort in analyzing the program. It would be nice if there was a simple way to get closer to this security without requiring as much effort.

Clean up that Pod and Service before moving to the next section:

$ "kubectl delete service fine-pod --wait"
$ "kubectl delete pod fine-pod --wait --now"

---

**Create Pod that uses the container runtime default seccomp profile**

Most container runtimes provide a sane set of default syscalls that are allowed or not. You can adopt these defaults for your workload by setting the seccomp type in the security context of a pod or container to RuntimeDefault.

![image](https://user-images.githubusercontent.com/88305831/177988511-a3a2293e-3692-477a-aa9c-09451a16f4f7.png)

Create that Pod:

$ "kubectl apply -f default-pod.yam"
$ "kubectl get pod default-pod -o wide"

![image](https://user-images.githubusercontent.com/88305831/177988932-78e6bf7f-ccc3-431c-9c49-c619f44c98eb.png)


Finally, now that you saw that work OK, clean up:

$ "kubectl delete pod default-pod --wait --now"





