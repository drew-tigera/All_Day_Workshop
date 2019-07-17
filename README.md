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
Verify the new namespace with the following command:
```
kubectl describe namespace dmz
```
3. Spin up an nginx deplyment in your default namespace. This should pull an IP from the original pod pool from when k8s was setup. 
```
kubectl create deployment nginx --image=nginx
```
Then run this command to find out the name of the nginx pod:
```
kubectl get pods
```
Note the full name of your pod and run this command (be sure to replace the bits in < > with your actual pod name:
```
kubectl describe pods <your_pod_name>
```
Scroll up to see the IP address assigned to your pod. It should be in the 192.168.0.0/16 subnet.

4. Spin up an nginx deployment in the DMZ namespace:
```
kubectl create deployment nginx --image=nginx -n dmz
```
Now run the same ```kubectl get pods``` & ```describe``` commands as in Step 3, just be sure to add ```-n dmz``` to each one.

You can also verify connectivity to the two subnets by ```curl <pod IPs>``` you should see the good old nginx default page.

5. Clean up 

```
kubectl delete deployment nginx
```
And then
```
kubectl delete ns dmz
```
And then
```
calicoctl delete ippool dmz-pool
```
And it's like it never happened. 

Lab 3: Stars Policy Tutorial

https://docs.projectcalico.org/v3.8/security/stars-policy/

Lab 4: Tigera Secure Tierd Policy Demo

https://docs.tigera.io/v2.4/security/tiered-policy

Lab 5: DNS Policy Lab: Create DNS based global network sets and then allow egress to them from a namespace

Step 1: Create a namespace named test
```
kubectl create ns test
```
Step 2: Create a DNS Global Network Set for tigera.io and google.com
```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: allowed-domains-1
  labels:
    external: allowed
spec:
  allowedEgressDomains:
  - tigera.io
  - google.com
EOF
```
Run ```kubectl describe globalnetworkset allowed-domains-1``` to verify your settings.

Step 2: Verify connectivity and setup your busybox access pod

Spin up a busybox access pod to test access
```
kubectl run --namespace=test access --rm -ti --image busybox /bin/sh
```
Try tigera.io with wget
```
wget -q --timeout=5 tigera.io -O -
```
Leave this terminal window open and start a new one for the next steps.

Step 3: Block egress traffic in the test namespace - first we have to block everything so that we can subsequently open what we want back up.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: test
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
EOF
```
Step 4: Verify egress blocked

Use your busybox terminal window to test connectivity to google.com and tigera.io
```
wget -q --timeout=5 google.com -O -
```

Step 5: Allow DNS egress - we got to let it look up the domain name somehow

First we're going to add a label to our kube-system namespace which is where the DNS system of K8s, coredns, is running.
```
kubectl label namespace kube-system name=kube-system
```
Then we will create a policy to allow DNS communication from the test namespace to coredns in the kube-system namespace
```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-access
  namespace: test
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF
```

Step 6: Yes finally now we can create the allowed DNS egress policy for the test namespace.
```
kubectl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-egress-to-domain
spec:
  order: 1
  selector: projectcalico.org/namespace == 'test'
  types:
  - Egress
  egress:
  - action: Allow
    destination:
      selector: external == 'allowed'
EOF
```


