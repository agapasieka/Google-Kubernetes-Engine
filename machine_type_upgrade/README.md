<!-- Overview -->
## Overview
In this lab, you will deploy a GKE cluster with a 'Hello World!' application and upgrade the machine type of the node pool while keeping the workloads up and running with zeor downtime. 

<!-- Task1 -->
## Enable the Kubernetes Engine API
```sh
gcloud services enable container.googleapis.com
  ```

<!-- Task2 -->
## Set the environment variable for the zone and cluster name
  ```sh
export REGION=
export CLUSTER=
  ```

<!-- Task3 -->
## Create a Kubernetes cluster
  ```sh
gcloud container clusters create $CLUSTER --num-nodes 1 --region $REGION --machine-type=e2-medium --enable-ip-alias
  ```  

<!-- Task4 -->
## Deploy sample Hello app
  ```sh
kubectl apply -f hello-app.yaml 
  ```

## Verify the node pool, nodes, and pods are running
  ```sh
kubectl get nodes
kubectl get pods -o=wide
  ``` 

## Verify the hello application is running
  ```sh
kubectl get services
  ```

<!-- Task5 -->
## Create the new node pool with a larger machine type
  ```sh
gcloud container node-pools create larger-pool --cluster=sample-cluster --num-nodes=1 --machine-type=e2-highmem-2
  ```

## Verify the new pool and nodes are ready
 ```sh
gcloud container node-pools list --cluster $CLUSTER --region $REGION
kubectl get nodes -l cloud.google.com/gke-nodepool=larger-pool
 ```

<!-- Task6 -->
## Migrating the application workloads
This task will require the following steps:
* Cordon the existing node pool - this will mark the nodes in the current pool as 'unschedulable' so GKE stops assigning new pods to these nodes
* Drain the existing node pool - this will evict the workloads running in the existing node pool gracefully without downtime

## Cordon the existing node pool
  ```sh
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
     kubectl cordon "$node";
  done
  ``` 

## Drain the existing node pool
  ```sh
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
  kubectl drain --force --ignore-daemonsets --delete-emptydir-data --grace-period=10  "$node";
done

  ```

## Verify the nodes in the existing node pool have SchedulingDisabled status in the node list
 ```sh
kubectl get nodes  
 ```   

## View the pods are now running on the nodes in the new node pool
  ```sh
kubectl get pods -o=wide
  ```

Verify in browser if the hello app is still running


<!-- Task7 -->
## Delete created resources 
  ```sh
kubectl delete -f deployment hello-app
  ```

## Delete cluster
```sh
gcloud container clusters delete $CLUSTER --region $REGION
```


## The End
