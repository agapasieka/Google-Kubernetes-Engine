# Overview

In this lab, you will perform the following tasks:

* Observe Kubernetes DNS in action.
* Define various service types (ClusterIP, NodePort, LoadBalancer) in manifests along with label selectors to connect to existing labeled Pods and deployments, deploy those to a cluster, and test connectivity.
* Deploy an Ingress resource that connects clients to two different services based on the URL path entered.
* Verify Google Cloud network load balancer creation for type=LoadBalancer services.

Perform the tasks in CLoud Shell or use your favourite terminal. 

## Set the environment variable for the zone and cluster name
```
export ZONE=
export CLUSTER=
```

## Create a Kubernetes cluster
```
gcloud container clusters create $CLUSTER --num-nodes 3 --zone $ZONE --enable-ip-alias
```  

## Configure access to your cluster
```
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```

## Create Pods and services to test DNS resolution
Create a service called dns-demo with two sample application Pods called dns-demo-1 and dns-demo-2.
```
cat > dns-demo.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: dns-demo
spec:
  selector:
    name: dns-demo
  clusterIP: None
  ports:
  - name: dns-demo
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: dns-demo-1
  labels:
    name: dns-demo
spec:
  hostname: dns-demo-1
  subdomain: dns-demo
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: dns-demo-2
  labels:
    name: dns-demo
spec:
  hostname: dns-demo-2
  subdomain: dns-demo
  containers:
  - name: nginx
    image: nginx
```
Ctrl + C to exit

## Create the service and Pods
```
kubectl apply -f dns-demo.yaml
```
## Verify that your Pods are running
```
kubectl get pods
```
## To get inside the cluster, open an interactive session to bash running from dns-demo-1
```
kubectl exec -it dns-demo-1 -- /bin/bash
```
## Update apt-get and install a ping tool
```
apt-get update
apt-get install -y iputils-ping
```
## Ping dns-demo-2
```
ping dns-demo-2.dns-demo.default.svc.cluster.local
```
This ping should succeed and report that the target has the ip-address you found earlier for the dns-demo-2 Pod.
Press Ctrl+C to abort the ping command.

## Ping the dns-demo service's FQDN, instead of a specific Pod inside the service
```
ping dns-demo.default.svc.cluster.local
```
This ping should also succeed but it will return a response from the FQDN of one of the two demo-dns Pods. This Pod might be either demo-dns-1 or demo-dns-2.
Press Ctrl+C to abort the ping command but leave the interactive shell on dns-demo-1 running.

## Deploy a sample workload and a ClusterIP service
Create **hello-v1.yaml** manifest file
```
cat > hello-v1.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      run: hello-v1
  template:
    metadata:
      labels:
        run: hello-v1
        name: hello-v1
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        name: hello-v1
        ports:
        - containerPort: 8080
          protocol: TCP
```
Ctrl + C to exit

## Start a new Cloud Shell session and create a deployment from the hello-v1.yaml
```
kubectl create -f hello-v1.yaml
```
## View a list of deployments
```
kubectl get deployments
```
## Deploy a Service using a ClusterIP 
Create manifest file called **hello-svc.yaml**
```
cat > hello-svc.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
spec:
  type: ClusterIP
  selector:
    name: hello-v1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```
Ctrl + C to exit

## Deploy the ClusterIP service
```
kubectl apply -f hello-svc.yaml
```
This manifest defines a ClusterIP service and applies it to Pods that correspond to the selector with the **name: hello-v1** label.

## Verify that the service was created and that a Cluster-IP was allocated
```
kubectl get service hello-svc
```

## Test your application from the cloud shell, outside the cluster
```
curl hello-svc.default.svc.cluster.local
```
The connection should fail because that service is not exposed outside of the cluster.

## Return to your first Cloud Shell window with interractive session to dns-demo-1 Pod and install curl so you can make calls to web services from the command line
```
apt-get install -y curl
```
## Test the HTTP connection between the Pods
```
curl hello-svc.default.svc.cluster.local
```
This connection should succeed. This connection works because the clusterIP can be resolved using the internal DNS within the Kubernetes Engine cluster.

## Convert the service to use NodePort
Create a file called **hello-nodeport-svc.yaml** 
```
cat > hello-nodeport-svc.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
spec:
  type: NodePort
  selector:
    name: hello-v1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30100
```
Ctrl + C to exit

## Deploy the manifest that changes the service type for the hello-svc to NodePort
```
kubectl apply -f hello-nodeport-svc.yaml
```
## Verify that the service type has changed to NodePort
```
kubectl get service hello-svc
```

## Test your application from the cloud shell, outside the cluster
```
curl hello-svc.default.svc.cluster.local
```
The connection should fail because that service is not exposed outside of the cluster.

## Return to your first Cloud Shell window with interractive session to dns-demo-1 Pod and test the HTTP connection between the Pods
```
curl hello-svc.default.svc.cluster.local
```
This connection should succeed. 

## Reserve Static Regional IP address for Load Balancer Service
```
export REGION=
gcloud compute addresses create regional-loadbalancer --region=$REGION
```
## Deploy a new set of Pods and a LoadBalancer service
Create a second Deployment manifest called **hello-v2.yaml** that runs version 2 of the sample hello application on port 8080.
A different version of the application allows easily differentiate the two services. 
```
cat > hello-v2.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-v2
spec:
  replicas: 3
  selector:
    matchLabels:
      run: hello-v2
  template:
    metadata:
      labels:
        run: hello-v2
        name: hello-v2
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:2.0
        name: hello-v2
        ports:
        - containerPort: 8080
          protocol: TCP
```
##  Deploy the manifest file hello-v2.yaml
```
kubectl create -f hello-v2.yaml
```
## View a list of deployments
```
kubectl get deployments
```

## Deploy a LoadBalancer Service using the hello-lb-svc.yaml
Create the manifest file hello-lb-svc.yaml
```
cat > hello-lb-svc.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: hello-lb-svc
spec:
  type: LoadBalancer
  loadBalancerIP: 10.10.10.10
  selector:
    name: hello-v2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

## Save the regional static IP address you created earlier into an environment variable
```
export STATIC_LB=$(gcloud compute addresses describe regional-loadbalancer --region REGION --format json | jq -r '.address')
```
## Replace the placeholder address with the regional static IP-address
```
sed -i "s/10\.10\.10\.10/$STATIC_LB/g" hello-lb-svc.yaml
```
## Deploy your LoadBalancer service manifest
```
kubectl apply -f hello-lb-svc.yaml
```
## Verify that the service was created
```
kubectl get services
```
In GKE environments the external load balancer for the LoadBalancer service type is implemented using a Google Cloud load balancer and will take a few minutes to create. This external IP address makes the service accessible from outside the cluster.

## Test your application
In the Cloud Shell, attempt to open an HTTP session to the new service
```
curl hello-lb-svc.default.svc.cluster.local
```
The connection should fail because that service name is not exposed outside of the cluster and the external IP address is not registered with this hostname.

## Test connection again using the External IP address associated with the Regional Load Balancer service. 
```
curl $STATIC_LB
```
This connection should succeed as the LoadBalancer's external ip-address can be reached from outside Google Cloud.

## Return to your first Cloud Shell window with interractive session to dns-demo-1 Pod and test the HTTP connection between the Pods
```
curl hello-lb-svc.default.svc.cluster.local
```
This connection should succeed. 

## Deploy an ingress resource
We have two services in the cluster for the hello application. One service is hosting version 1.0 via a NodePort service, while the other service is hosting version 2.0 via a LoadBalancer service. You will now deploy an Ingress resource that will direct traffic to both services based on the URL entered by the user.

In GKE, Ingress is implemented using Cloud Load Balancing. When you create an ingress resource in your cluster, GKE creates an HTTP(S) load balancer and configures it to route traffic to your application.

## Reserve Static Global IP for Ingress Resource
```
gcloud compute addresses create global-ingress --global
```

## Create a sample manifest called **hello-ingress.yaml**
```
cat > hello-ingress.yaml
```
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.global-static-ip-name: "global-ingress"
spec:
  rules:
  - http:
      paths:
      - path: /v1
        pathType: ImplementationSpecific
        backend:
          service:
            name: hello-svc
            port:
              number: 80
      - path: /v2
        pathType: ImplementationSpecific
        backend:
          service:
            name: hello-lb-svc
            port:
              number: 80
```
This configuration file defines an ingress resource that will attach to the global-ingress static ip address and create a global HTTP(S) load balancer that directs traffic to the web NodePort service and the LoadBalancer service that we exposed.

## Deploy the manifest
```
kubectl apply -f hello-ingress.yaml
```

## Test your application
Get the external IP address of the load balancer serving your application
```
kubectl describe ingress hello-ingress
```
## Test the application using the External IP address associated with the Ingress resource. Be sure to include the /v1 in the URL path
```
curl http://[external_IP]/v1
```
The v1 URL is configured in hello-ingress.yaml to point to the hello-svc NodePort service that directs traffic to the v1 application Pods.

## Now test the v2 URL path from Cloud Shell.
```
curl http://[external_IP]/v2
```
The v2 URL is configured in hello-ingress.yaml to point to the hello-lb-svc LoadBalancer service that directs traffic to the v2 application Pods.

## Delete created resources
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End





























