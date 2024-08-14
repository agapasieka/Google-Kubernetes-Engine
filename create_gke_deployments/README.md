<!-- Overview -->
## Overview
In this lab, you learn how to perform the following tasks:

* Create deployment manifests, deploy to cluster, and verify Pod rescheduling as nodes are disabled.
* Trigger manual scaling up and down of Pods in deployments.
* Trigger deployment rollout (rolling update to new version) and rollbacks.
* Perform a Canary deployment.

Perform steps in cloud shell or terminal of your choice. 

<!-- Task1 -->
## Set the environment variable for the zone and cluster name
  ```sh
export REGION=
export CLUSTER=
  ```

<!-- Task2 -->
## Create a Kubernetes cluster

  ```sh
  gcloud container clusters create $CLUSTER --num-nodes 3 --region $REGION --enable-ip-alias
  ```  

<!-- Task3 -->
## Configure access to your cluster
  ```sh
  gcloud container clusters get-credentials $CLUSTER --region $REGION
  ```

<!-- Task4 -->
## Create a sample deployment manifest called nginx-deployment.yaml
This deployment is configured to run three Pod replicas with a single nginx container in each Pod listening on TCP port 80.
  ```sh
cat > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
  ``` 

<!-- Task5 -->
## Deploy the manifest
  ```sh
kubectl apply -f ./nginx-deployment.yaml
  ```

<!-- Task6 -->
## View a list of deployments
  ```sh
  kubectl get deployments
  ``` 

<!-- Task7 -->
## Scale udown the number of Pods in deployments
  ```sh
kubectl scale --replicas=1 deployment nginx-deployment
  ```

<!-- Task8 -->
## Trigger a deployment rollout by updating the version of nginx in the deployment
 ```sh
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1 
 ```   

<!-- Task9 -->
## Annotate the rollout with details on the change
  ```sh
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="version change to 1.9.1" --overwrite=true
  ```

<!-- Task10 -->
## View the rollout status
  ```sh
kubectl rollout status deployment.v1.apps/nginx-deployment   
  ```

<!-- Task11 -->
## Verify the change
 ```sh
kubectl get deployments   
 ```

<!-- Task12 -->
## View the rollout history of the deployment
 ```sh
kubectl rollout history deployment nginx-deployment 
  ``` 

<!-- Task13 -->
## Trigger a deployment rollback
 ```sh
kubectl rollout undo deployments nginx-deployment
  ```

<!-- Task14 -->
## View the updated rollout history of the deployment 
  ```sh
kubectl rollout history deployment nginx-deployment   
  ```

<!-- Task15 -->
## View the details of the latest deployment revision
  ```sh
kubectl rollout history deployment/nginx-deployment --revision=3  
  ```

<!-- Task16 -->
## Define the service type in the manifest
In this task, you create and verify a service that controls inbound traffic to an application. Services can be configured as ClusterIP, NodePort or LoadBalancer types. 
Create manifest file called service-nginx.yaml that deploys a LoadBalancer service type. 
  ```sh
cat > service-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 60000
    targetPort: 80 
  ```
This service is configured to distribute inbound traffic on TCP port 60000 to port 80 on any containers that have the label app: nginx.

<!-- Task17 -->
## Deploy your manifest
  ```sh
kubectl apply -f ./service-nginx.yaml
  ```

<!-- Task17 -->
## Verify the LoadBalancer creation
  ```sh
kubectl get service nginx
  ```
Output:

NAME      CLUSTER_IP      EXTERNAL_IP      PORT(S)   SELECTOR    AGE
nginx     10.X.X.X        X.X.X.X          60000/TCP    run=nginx   1m

<!-- Task18 -->
## Create a canary deployment file called nginx-canary.yaml to deploy a single pod running a newer version of nginx than your main deployment.
  ```sh
cat > nginx-canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-canary
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        track: canary
        Version: 1.9.1
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
  ```

<!-- Task19 -->
## Create the canary deployment based on the configuration file
  ```sh
kubectl apply -f ./nginx-canary.yaml
  ```

<!-- Task20 -->
## Verify that both the nginx and the nginx-canary deployments are present
  ```sh
kubectl get deployments
  ```

<!-- Task21 -->
## Scale down the primary deployment to 0 replicas
  ```sh
kubectl scale --replicas=0 deployment nginx-deployment
  ```

<!-- Task22 -->
## Verify that the only running replica is now the Canary deployment
  ```sh
kubectl get deployments
  ```

<!-- Task23 -->
## Set the sessionAffinity field to ClientIP in the specification of the service if you need a client's first request to determine which Pod will be used for all subsequent connections.
  ```sh
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 60000
    targetPort: 80
  ```


<!-- Task24 -->
## Delete created resources 
  ```sh
kubectl delete -f ./nginx-deployment.yaml
kubectl delete -f ./service-nginx.yaml
kubectl delete -f ./nginx-canary.yaml
  ```
## Delete cluster
```sh
gcloud container clusters delete $CLUSTER --region $REGION
```


## The End
