<!-- Overview -->
## Overview

In this lab, you set up an application in Google Kubernetes Engine (GKE), and then use a HorizontalPodAutoscaler to autoscale the web application. You then work with multiple node pools of different types, and you apply taints and tolerations to control the scheduling of Pods with respect to the underlying node pool.




<!-- Task1 -->
## Set the environment variable for the zone and cluster name
  ```sh
export ZONE=
export CLUSTER=
  ```

<!-- Task2 -->
## Create a Kubernetes cluster
  ```sh
gcloud container clusters create $CLUSTER --num-nodes 3 --zone $ZONE --enable-ip-alias
  ```  

<!-- Task3 -->
## Configure access to your cluster
  ```sh
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
  ```

<!-- Task4 -->
## Deploy a sample web application to your GKE cluster
  ```sh
kubectl create -f web.yaml --save-config
  ``` 

<!-- Task5 -->
## Create a service resource of type NodePort on port 8080 for the web deployment 
  ```sh
kubectl expose deployment web --target-port=8080 --type=NodePort
  ```

<!-- Task5 -->
## Verify that the service was created and that a node port was allocated
  ```sh
kubectl get service web  
  ``` 

<!-- Task6 -->
## Configure autoscaling
1. Get the list of deployments to determine whether your sample web application is still running
  ```sh
kubectl get deployment
  ```

2. Configure autoscaling with a CPU utilization target of 1%
 ```sh
kubectl autoscale deployment web --max 4 --min 1 --cpu-percent 1  
 ```   
The kubectl autoscale command creates a HorizontalPodAutoscaler object that targets a specified resource, called the scale target, and scales it as needed.

3. Get the list of HorizontalPodAutoscaler resources
  ```sh
kubectl get hpa   
  ```

##  Inspect the configuration of HorizontalPodAutoscaler in table form
  ```sh
kubectl describe horizontalpodautoscaler web   
  ```

<!-- Task7 -->
## Test the autoscale configuration
You need to create a heavy load on the web application to force it to scale out.

1. Create the loadgen.yaml file
 ```sh
cat > loadgen.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loadgen
spec:
  replicas: 4
  selector:
    matchLabels:
      app: loadgen
  template:
    metadata:
      labels:
        app: loadgen
    spec:
      containers:
      - name: loadgen
        image: k8s.gcr.io/busybox
        args:
        - /bin/sh
        - -c
        - while true; do wget -q -O- http://web:8080; done
 ```

2. Deploy the container
 ```sh
kubectl apply -f loadgen.yaml 
  ``` 

3. Get the list of deployments to verify that the load generator is running
 ```sh
kubectl get deployment
  ```

4. After a few minutes, inspect HorizontalPodAutoscaler 
  ```sh
kubectl get hpa   
  ```

5. To stop the load on the web application, scale the loadgen deployment to zero replicas
  ```sh
kubectl scale deployment loadgen --replicas 0  
  ```

6. Wait 2 to 3 minutes and verify that the web application has scaled down to the minimum value of 1 replica 
  ```sh
kubectl get deployment
  ```

<!-- Task7 -->
## Manage node pools
1. Deploy a new node pool with two preemptible VM instances
 ```sh
gcloud container node-pools create "temp-pool-1" \
--cluster=$my_cluster --zone=$my_zone \
--num-nodes "2" --node-labels=temp=true --preemptible
  ```

2. Get the list of nodes to verify that the new nodes are ready
  ```sh
kubectl get nodes
  ```

3. List only the nodes with the temp=true label
  ```sh
kubectl get nodes -l temp=true
  ```

To prevent the scheduler from running a Pod on the temporary nodes, you add a taint to each of the nodes in the temp pool. Taints are implemented as a key-value pair with an effect (such as NoExecute) that determines whether Pods can run on a certain node. Only nodes that are configured to tolerate the key-value of the taint are scheduled to run on these nodes.

4. Add a taint to each of the newly created nodes
  ```sh
kubectl taint node -l temp=true nodetype=preemptible:NoExecute
  ```

5. Allow application Pods to execute on these tainted nodes, by editing the web.yaml file and adding a tolerations key to the deployment configuration.
 ```sh
tolerations:
- key: "nodetype"
  operator: Equal
  value: "preemptible"
 ```

The spec section of file should look like: 
```sh
...
    spec:
      tolerations:
      - key: "nodetype"
        operator: Equal
        value: "preemptible"
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          # You must specify requests for CPU to autoscale
          # based on CPU utilization
          requests:
            cpu: "250m"
```
6. Force the web deployment to use the new node-pool by adding a nodeSelector key in the template's spec section. 
 ```sh
     nodeSelector:
        temp: "true"
 ```

7. The full web.yaml deployment should now look as follows:
 ```sh
 apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      run: web
  template:
    metadata:
      labels:
        run: web
    spec:
      tolerations:
      - key: "nodetype"
        operator: Equal
        value: "preemptible"
      nodeSelector:
        temp: "true"
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          # You must specify requests for CPU to autoscale
          # based on CPU utilization
          requests:
            cpu: "250m"
 ```

8.  Apply this change
 ```sh
kubectl apply -f web.yaml
 ```

9. Confirm the change by inspecting the running web Pod(s)
 ```sh
kubectl describe pods -l run=web
 ```

10. Force the web application to scale out again, scale the loadgen deployment back to four replicas
 ```sh
kubectl scale deployment loadgen --replicas 4
 ```

11. Get the list of Pods using the wide output format to show the nodes running the Pods
 ```sh
kubectl get pods -o wide
 ```
This shows that the loadgen app is running only on default-pool nodes while the web app is running only the preemptible nodes in temp-pool-1

<!-- Task8 -->

## Delete created resources 
  ```sh
kubectl delete -f deployment web
kubectl delete -f deployment loadgen
  ```
## Delete cluster
```sh
gcloud container clusters delete $CLUSTER --zone $ZONE
```
## The End
