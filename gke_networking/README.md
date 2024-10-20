# Overview
You have several options to lock down your cluster to varying degrees:

* The whole cluster can have external access.
* The whole cluster can be private.
* The nodes can be private while the cluster control plane is public, and you can limit which external networks are authorized to access the cluster control plane.

In this lab, you will perform the following tasks:

* Create and test a private cluster.
* Configure a cluster for authorized network control plane access.
* Configure a network policy for Pod security.

## Create a private cluster
In a private cluster, the nodes have internal IPs only, which ensures that their workloads are isolated from the public Internet.

1. Set the environment variable for the zone and cluster name
   ```
   export ZONE=
   export CLUSTER=
   ```
2. Deploy the cluster
   ```
   gcloud container clusters create $CLUSTER \
    --num-nodes 3 \
    --zone $ZONE \
    --enable-ip-alias \
    --enable-private-nodes \
    --enable-private-endpoint

   ```

## Add an authorized network for cluster control plane access
1. Find out Cloud Shell IP and save it to variable.
   ```
   export AUTHORIZED_NETWORK=$(curl -s ifconfig.me)
   ```
2. Update the cluster
   ```
   gcloud container clusters update $CLUSTER \
    --zone $ZONE \
    --enable-master-authorized-networks \
    --master-authorized-networks $AUTHORIZED_NETWORK
   ```

## Create a cluster network policy
A cluster network policy will restrict communication between the Pods. 

1. Enable cluster network policies on the cluster
   ```
   gcloud container clusters update $CLUSTER \
    --zone $ZONE \
    --enable-network-policy
   ```
2. Connect to a GKE cluster
   ```
   gcloud container clusters get-credentials $CLUSTER --zone $ZONE
   ```
3. Run a simple web server application with the label app=hello, and expose the web application internally in the cluster
   ```
   kubectl run hello-web --labels app=hello --image=gcr.io/google-samples/hello-app:1.0 --port 8080 --expose
   ```

## Restrict incoming traffic to Pods
Create a Network Policy manifest file, allowing ingress traffic to Pods labeled **app: hello** from Pods labeled **app: test**

1. Create a file called **hello-allow-from-test.yaml**
   ```
   cat > hello-allow-from-test.yaml
   ```
2. Paste the following content
   ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: hello-allow-from-test
    spec:
      policyTypes:
      - Ingress
      podSelector:
        matchLabels:
          app: hello
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: test
   ```
Ctrl + C to exit

3. Create an ingress policy
  ```
  kubectl apply -f hello-allow-from-test.yaml
  ```
4. Verify that the policy was created
   ```
   kubectl get networkpolicy
   ```

## Validate the ingress policy
1. Run a temporary Pod called test-1 with the label app=test and get a shell in the Pod
   ```
   kubectl run test-1 --labels app=test --image=alpine --restart=Never --rm -it
   ```
2. Make a request to the hello-web:8080 endpoint to verify that the incoming traffic is allowed
   ```
   wget -qO- --timeout=2 http://hello-web:8080
   ```
3. Type exit and press ENTER to leave the shell


4. Now you will run a different Pod using the same Pod name but using a label, app=other, that does not match the podSelector in the active network policy. This Pod should not have the ability to access the hello-web application
  ```
  kubectl run test-1 --labels app=other --image=alpine --restart=Never --rm -it
  ```
5. Make a request to the hello-web:8080 endpoint to verify that the incoming traffic is not allowed
   ```
   wget -qO- --timeout=2 http://hello-web:8080
   ```
The request times out.


## Restrict outgoing traffic from the Pods
You can control outgoing (egress) traffic the same way you control incoming traffic. However, if you want to access internal services (like hello-web) or external websites (like www.example.com), you need to allow DNS traffic in your egress policies. 
DNS uses port 53 and works over both TCP and UDP.

1. Create Egrees Network Policy file **test-allow-to-hello.yaml** that permits Pods with the label **app: test** to communicate with Pods labeled **app: hello** on any port number, and allows the Pods labeled **app: test** to communicate to any computer on UDP port 53, which is used for DNS resolution.
   Without the DNS port open, you will not be able to resolve the hostnames.
   ```
   cat > test-allow-to-hello.yaml
   ```
2. Paste the following content
   ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: test-allow-to-hello
    spec:
      policyTypes:
      - Egress
      podSelector:
        matchLabels:
          app: test
      egress:
      - to:
        - podSelector:
            matchLabels:
              app: hello
      - to:
        ports:
        - protocol: TCP
          port: 53
        - protocol: UDP
          port: 53

    ```
   Ctrl + C to exit

 3. Create an egress policy
    ```
    kubectl apply -f test-allow-to-hello.yaml
    ```
4. Verify that the policy was created
   ```
   kubectl get networkpolicy
   ```

## Validate the egress policy
1. Deploy a new web application called hello-web-2 and expose it internally in the cluster
   ```
   kubectl run hello-web-2 --labels app=hello-2 --image=gcr.io/google-samples/hello-app:1.0 --port 8080 --expose
   ```
2. Run a temporary Pod with the **app=test-2** label and get a shell prompt inside the container
   ```
   kubectl run test-2 --labels app=test-2 --image=alpine --restart=Never --rm -it
   ```
3. Verify that the Pod can establish connections to hello-web:8080
   ```
   wget -qO- --timeout=2 http://hello-web:8080
   ```
4. Verify that the Pod cannot establish connections to hello-web-2:8080
   ```
   wget -qO- --timeout=2 http://hello-web-2:8080
   ```
   This fails because none of the Network policies you have defined allow traffic to Pods labeled **app: hello-2**

5. Verify that the Pod cannot establish connections to external websites, such as www.example.com
  ```
  wget -qO- --timeout=2 http://www.example.com
  ```
This fails because the network policies do not allow external http traffic (tcp port 80).
Type exit and press ENTER to leave the shell.

## Delete created resources
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End
