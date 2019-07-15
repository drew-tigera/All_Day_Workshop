# All_Day_Workshop

Welcome everyone! Ask your instructor if you haven't already got a lab server sheet to do the labs.

Lab 1 - Getting to know your lab environment

Steps:
1: Open up an SSH client on your laptop and SSH into your training server.
  ```
    ssh -i nycclass.pem centos@<IP_address_of_your_training_server>
  ```
Look on your lab sheet for a link to download the above PEM file if you don't already have it. 

2. In the SSH session on your training server run the following commands:
```
  kubectl get nodes
```
   You should see there's just one combo node for your training server. This is an unusual way to run Kubernetes, but it will work just fine for our labs.
```
   kubectl get pods --namespace kube-system
```
   You can see that we have Tigera Secure running on top of our Kubernetes install. We will be using Tigera Secure and Calico to lock down and secure our demo workloads on Kubernetes.


