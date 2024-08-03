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

4. 
  ```sh
   
  ```

5. 
 ```sh
   
 ```

<!-- Task3 -->
## 
1. 
 ```sh
 
  ``` 

2. 
 ```sh

  ```

3.  
  ```sh
   
  ```

4. 
  ```sh
  
  ```



## The End
