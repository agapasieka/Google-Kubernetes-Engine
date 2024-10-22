# Overview
Configure and test Kubernetes liveness and readiness probes for containers.

In this lab, you will perform the following tasks:

* Configure and test a liveness probe
* Configure and test a readiness probe

## Set the environment variable for the zone and cluster name
```
export ZONE=
export CLUSTER=
```
## Create a GKE cluster
```
gcloud container clusters create $CLUSTER --num-nodes 2 --zone $ZONE --enable-ip-alias --logging=SYSTEM --monitoring=SYSTEM
```
## Configure access to your cluster
```
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```

## Configure a liveness probe
Deploy a liveness probe to detect applications that have transitioned from a running state to a broken state. 
Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup.

In such cases, you don't want to kill the application, but you don't want to send it requests either. 
Kubernetes provides readiness probes to detect and mitigate these situations. 
A Pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

Readiness probes are configured similarly to liveness probes. 
The only difference is that you use the readinessProbe field instead of the livenessProbe field.

Create a Pod definition file called exec-liveness.yaml, that defines a simple container called liveness running Busybox and a liveness probe that uses the cat command against the file /tmp/healthy within the container to test for liveness every 5 seconds.
The startup script for the liveness container creates the /tmp/healthy on startup and then deletes it 30 seconds later to simulate an outage that the Liveness probe can detect.

## Create exec-liveness.yaml manifest
```
cat > exec-liveness.yaml
```
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
Ctrl + C to exit

## Create the Pod
```
kubectl create -f exec-liveness.yaml
```
## View the Pod events
```
kubectl describe pod liveness-exec
```
At the bottom of the output, there are messages indicating that the liveness probes have failed, and the containers have been killed and recreated.
Wait  60 seconds, and verify that the Container has been restarted
```
kubectl get pod liveness-exec
```
The output shows that RESTARTS has been incremented in response to the failure detected by the liveness probe.

# Configure readiness probes
Although a pod could successfully start and be considered healthy by a liveness probe, it's likely that it may not be ready to receive traffic right away. This is common for deployments that serve as a backend to a service such as a load balancer. A readiness probe is used to determine when a pod and its containers are ready to begin receiving traffic.

Readiness probes are configured similarly to liveness probes. The only difference in configuration is that you use the readinessProbe field instead of the livenessProbe field. Readiness probes control whether a specific container is considered ready, and this is used by services to decide when a container can have traffic sent to it.

Create a Pod definition file called readiness-deployment.yaml that create a single pods that will serve as a test web server along with a load balancer. 

The container has a readiness probe defined that uses the cat command against the file /tmp/healthz within the container to test for readiness every 5 seconds.

Each container also has a liveness probe defined that uses the cat command against the same file within the container to test for readiness every 5 seconds but it also has a startup delay of 45 seconds to simulate an application that might have a complex startup process that takes time to stabilize after a container has started.

Once the service has started handling traffic this pattern ensures that the service will forward traffic only to containers that are ready to handle traffic.

## Create readiness-deployment.yaml manifest
```
cat > readiness-deployment.yaml
```
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    demo: readiness-probe
  name: readiness-demo-pod
spec:
  containers:
  - name: readiness-demo-pod
    image: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthz
      initialDelaySeconds: 5
      periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo-svc
  labels:
    demo: readiness-probe
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    demo: readiness-probe
```
Ctrl + C to exit

## Create the Pod
```
kubectl create -f readiness-deployment.yaml
```
## Check the status of the readiness-demo-svc load balancer service
```
kubectl get service readiness-demo-svc
```
Enter the IP address in a browser window and you'll notice that you'll get an error message signifying that the site cannot be reached

## Check the pod's events
```
kubectl describe pod readiness-demo-pod
```
The output will reveal that the readiness probe has failed
Unlike the liveness probe, an unhealthy readiness probe does not trigger the pod to restart.

## Generate the file that the readiness probe is checking for
```
kubectl exec readiness-demo-pod -- touch /tmp/healthz
```
The Conditions section of the pod description should now show True as the value for Ready
```
kubectl describe pod readiness-demo-pod | grep ^Conditions -A 5
```
Refresh the browser tab that had your readiness-demo-svc external IP. You should see a "Welcome to nginx!" message properly displayed.


The combination of the liveness and readiness probes provides a way to ensure that failed systems are restarted safely while the service only forwards traffic on to containers that are known to be able to respond.

## Delete created resources
```
kubectl delete -f exec-liveness.yaml
kubectl delete -f readiness-deployment.yaml
```
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End    
