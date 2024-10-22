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

