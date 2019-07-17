# All_Day_Workshop

Welcome everyone! Ask your instructor if you haven't already got a lab server sheet to do the labs.

Lab 1 - Getting to know your lab environment

Steps:

1: Open up an SSH client on your laptop and SSH into your training server.
  ```
    ssh -i nycclass.pem centos@<IP_address_of_your_training_server>
  ```
   Look on your lab sheet for a link to download the above PEM file if you don't already have it.
   
2. In the SSH session on your training server run the following command:
```
  kubectl get nodes
```
   You should see there's just one combo node for your training server. This is an unusual way to run Kubernetes, but it will work just fine for our labs.
```
   kubectl get pods --namespace kube-system
```
   You can see that we have Tigera Secure running on top of our Kubernetes install. We will be using Tigera Secure and Calico to lock down and secure our demo workloads on Kubernetes.
```
  calicoctl get license
```
   This will show the temporary license we have for Tigera Secure. As stated in the slides ```calicoctl``` is a tool very similar to ```kubectl``` but with certain specifc use cases when managing Calico and Tigera Secure. 

Lab 2 - Configuring the Calico CNI

Steps:

1: Go back to your SSH session we are going to setup a new subnet for our DMZ namespace. First we will create the IP Pool to use with the DMZ namespace

```
calicoctl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
   name: dmz-pool
spec:
   cidr: 10.0.1.0/24
   ipipMode: Always
   natOutgoing: true
EOF
```
Verify your pool with the following command:
```
calicoctl get ippools
``` 
You should see your new DMZ pool.

2. Create a Namespace that references the IP pool you just created:

```
kubectl create -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: dmz
  labels:
  	location: dmz
  annotations:
    "cni.projectcalico.org/ipv4pools": "[\"dmz-pool\"]"
EOF
```

   
 


