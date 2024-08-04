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
export ZONE=
export CLUSTER=
  ```

<!-- Task3 -->
## Create a Kubernetes cluster
  ```sh
  gcloud container clusters create $CLUSTER --num-nodes 3 --zone $ZONE --machine-type=e2-micro --enable-ip-alias
  ```  

<!-- Task4 -->
## Deploy sample Hello app
  ```sh
 
  ```

<!-- Task4 -->
## 
  ```sh

  ``` 

<!-- Task5 -->
## Deploy Pods to GKE clusters
  ```sh
kubectl create deployment --image nginx nginx-1
  ```

<!-- Task6 -->
## View all the deployed Pods in the active context cluster
  ```sh
kubectl get pods
  ```

<!-- Task7 -->
## Enter your Pod name into a variable
 ```sh
  export nginx_pod=
 ```

<!-- Task8 -->
## To be able to serve static content through the nginx web server, you must create and place a file into the container. 
  ```sh
  echo << EOF > test.html
<html> <header><title>This is title</title></header>
<body> Hello world </body>
</html>
EOF
  ``` 

<!-- Task9 -->
## Place the file into the appropriate location within the nginx container in the nginx Pod to be served statically
  ```sh
kubectl cp ~/test.html $nginx_pod:/usr/share/nginx/html/test.html
  ```

<!-- Task10 -->
## Create a service to expose our nginx Pod externally
 ```sh
  kubectl expose pod $nginx_pod --port 80 --type LoadBalancer
 ```   

<!-- Task11 -->
## View details about services in the cluster
  ```sh
   kubectl get services
  ```

<!-- Task12 -->
## Enter External IP into a variable
 ```sh
  export EXTERNAL_IP=
 ```

<!-- Task13 -->
## Verify that the nginx container is serving the static HTML file that you copied
  ```sh
   curl http://$EXTERNAL_IP/test.html
  ```

<!-- Task14 -->
## Delete created resources 
  ```sh
kubectl delete -f deployment nginx
  ```
## Delete cluster
```sh
gcloud container clusters delete $CLUSTER --zone $ZONE
```


## The End
