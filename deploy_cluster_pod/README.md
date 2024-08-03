<!-- Overview -->
## Overview
In this lab, you use the command line to build GKE cluster and deploy sample pod.

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
## Modify cluster to have four nodes
  ```sh
  gcloud container clusters resize $CLUSTER --zone $ZONE --num-nodes=4
  ```

<!-- Task4 -->
## Connect to a GKE cluster
  ```sh
  gcloud container clusters get-credentials $CLUSTER --zone $ZONE
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
