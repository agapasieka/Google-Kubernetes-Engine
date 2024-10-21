# Overview

In this lab, you will perform the following tasks:

* Create manifests for PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs) for Google Cloud persistent disks (dynamically created or existing)
* Mount Google Cloud persistent disk PVCs as volumes in Pods
* Use manifests to create StatefulSets
* Mount Google Cloud persistent disk PVCs as volumes in StatefulSets
* Verify the connection of Pods in StatefulSets to particular PVs as the Pods are stopped and restarted

**PersistentVolumes** are storage that is available to a Kubernetes cluster. 
**PersistentVolumeClaims** enable Pods to access PersistentVolumes. 
Without PersistentVolumeClaims Pods are mostly ephemeral, so you should use PersistentVolumeClaims for any data that you expect to survive Pod scaling, updating, or migrating.

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

## Create PVs and PVCs
Create a new file called **pvc-demo.yaml** that creates a 30 gigabyte PVC called hello-web-disk that can be mounted as a read-write volume on a single node at a time.
```
cat > pvc-demo.yaml
```
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hello-web-disk
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```
Ctrl + C to exit

## Create the PVC
```
kubectl apply -f pvc-demo.yaml
```
## Show your newly created PVC
```
kubectl get persistentvolumeclaim
```
Note: The status will remain pending until after the next step.

## Mount the PVC to a Pod
Create a manifest file pod-volume-demo.yaml to deploy an nginx container, attach the pvc-demo-volume to the Pod and mount that volume to the path /var/www/html inside the nginx container. 
Files saved to this directory inside the container will be saved to the persistent volume and persist even if the Pod and the container are shutdown and recreated.
```
cat > pod-volume-demo.yaml
```
```
kind: Pod
apiVersion: v1
metadata:
  name: pvc-demo-pod
spec:
  containers:
    - name: frontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: pvc-demo-volume
  volumes:
    - name: pvc-demo-volume
      persistentVolumeClaim:
        claimName: hello-web-disk
```

Ctrl + C to exit

## Create the Pod with the volume
```
kubectl apply -f pod-volume-demo.yaml
```
## List the Pods in the cluster
```
kubectl get pods
```
## Start the shell session to Pod to verify the PVC is accessible within the Pod
```
kubectl exec -it pvc-demo-pod -- sh
```
## Create simple web page in the Pod
```
echo Test webpage in a persistent volume!>/var/www/html/index.html
chmod +x /var/www/html/index.html
```
## Verify the web page
```
cat /var/www/html/index.html
```
## Exit the shell
```
exit
```
## Test the persistence of the PV
Delete the pvc-demo-pod
```
kubectl delete pod pvc-demo-pod
```
## List the Pods in the cluster
```
kubectl get pods
```
## Show the PVC
```
kubectl get persistentvolumeclaim
```
## Redeploy the pvc-demo-pod
```
kubectl apply -f pod-volume-demo.yaml
```
## List the Pods in the cluster
```
kubectl get pods
```
## Verify the PVC is still accessible within the Pod, start the shell session again
```
kubectl exec -it pvc-demo-pod -- sh
```
## Verify that the text file still contains your message
```
cat /var/www/html/index.html
```
## Exit shell session
```
exit
```

## Create StatefulSets with PVCs
A StatefulSet is like a Deployment, except that the Pods are given unique identifiers.
Before you can use the PVC with the statefulset, you must delete the Pod that is currently using it
```
kubectl delete pod pvc-demo-pod
```
## Create a StatefulSet called statefulset-demo.yaml that creates a StatefulSet that includes a LoadBalancer service and three replicas of a Pod containing an nginx container and a volumeClaimTemplate for 30 gigabyte PVCs with the name hello-web-disk. 
## The nginx containers mount the PVC called hello-web-disk at /var/www/html as in the previous task.
```
cat > statefulset-demo.yaml
```
```
  apiVersion: v1
  kind: Service
  metadata:
    name: statefulset-demo-service
  spec:
    ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
    type: LoadBalancer
  ---
  
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: statefulset-demo
  spec:
    selector:
      matchLabels:
        app: MyApp
    serviceName: statefulset-demo-service
    replicas: 3
    updateStrategy:
      type: RollingUpdate
    template:
      metadata:
        labels:
          app: MyApp
      spec:
        containers:
        - name: stateful-set-container
          image: nginx
          ports:
          - containerPort: 80
            name: http
          volumeMounts:
          - name: hello-web-disk
            mountPath: "/var/www/html"
    volumeClaimTemplates:
    - metadata:
        name: hello-web-disk
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 30Gi
```

Ctrl + C to exit

## Create the StatefulSet with the volume
```
kubectl apply -f statefulset-demo.yaml
```

## Verify the connection of Pods in StatefulSets
```
kubectl describe statefulset statefulset-demo
kubectl get pods
kubectl get pvc
```
## Verify the persistence of Persistent Volume connections to Pods managed by StatefulSets
Start the shell session to verify that the PVC is accessible within the Pod
```
kubectl exec -it statefulset-demo-0 -- sh
```
## Verify that there is no index.html text file in the /var/www/html directory
```
cat /var/www/html/index.html
```
## Create a simple text message as a web page in the Pod
```
echo Test webpage in a persistent volume!>/var/www/html/index.html
chmod +x /var/www/html/index.html
```
## Verify the text file contains your message
```
cat /var/www/html/index.html
```
## Exit the interactive shell
```
exit
```
## Delete the Pod where you updated the file on the PVC
```
kubectl delete pod statefulset-demo-0
```
## List the Pods in the cluster
```
kubectl get pods
```
You will see that the StatefulSet is automatically restarting the statefulset-demo-0 Pod.

## Connect to the shell on the new statefulset-demo-0 Pod
```
kubectl exec -it statefulset-demo-0 -- sh
```
## Verify that the text file still contains your messag
```
cat /var/www/html/index.html
```
## Exit the interactive shell
```
exit
```
## Delete created resources
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End

