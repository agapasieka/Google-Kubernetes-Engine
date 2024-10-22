# Overview

In this lab, you will perform the following tasks:

* Use IAM to control GKE access
* Create and use Pod security policies to control Pod creation
* Perform IP address and credential rotation

## Use IAM roles to grant administrative access to all the GKE clusters in the project
To allow a user to administer all GKE clusters in the project and to manage resources within those clusters, grand the **Kubernetes Engine Cluster Admin** role

To deploy a GKE cluster, a user must also be assigned the **iam.serviceAccountUser** role on the Compute Engine default service account.

## Define and use pod security admission
PodSecurity is a Kubernetes feature that helps you enforce security rules for Pods running in your GKE clusters. 
These rules, called Pod Security Standards, define different security levels, from permissive (allowing more freedom) to restrictive (locking down security tightly).

In this task, you'll create a security policy that allows only unprivileged Pods in the cluster's default namespace. 
Unprivileged Pods don’t let users run code as root and have limited access to the host system.

You'll also create a ClusterRole that can be used in a role binding. 
This will link the security policy to specific accounts that need to deploy unprivileged Pods.

For users who need to deploy privileged Pods, you can give them access to a built-in policy that allows admin users to do so, after enabling Pod Security Policies.

Once everything is set up, you’ll enable the PodSecurityPolicy controller to enforce these rules, and then test how they affect users with different permissions.

## Set the environment variable for the zone and cluster name
```
export ZONE=
export CLUSTER=
```
## Create a GKE cluster
```
gcloud container clusters create $CLUSTER --num-nodes 2 --zone $ZONE --enable-ip-alias
```
## Configure access to your cluster
```
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```
## Apply Pod Security Standards using PodSecurity

Create new namespaces:
* baseline-ns: For permissive workloads
* restricted-ns: For highly restricted workloads
```
 kubectl create ns baseline-ns
 kubectl create ns restricted-ns
```
## Use labels to apply security policies

Apply the following Pod Security Standards:

* baseline: Apply to baseline-ns in the warn mode
* restricted: Apply to restricted-ns in the enforce mode
```
 kubectl label --overwrite ns baseline-ns pod-security.kubernetes.io/warn=baseline
 kubectl label --overwrite ns restricted-ns pod-security.kubernetes.io/enforce=restricted
```
These commands achieve the following result:

* Workloads in the baseline-ns namespace that violate the baseline policy are allowed, and the client displays a warning message.
* Workloads in the restricted-ns namespace that violate the restricted policy are rejected, and GKE adds an entry to the audit logs.

## Test the configured policies
Deploy a workload that violates the baseline and the restricted policy to both namespaces, to verify that the PodSecurity admission controller works as intended.

Create a file called psa-workload.yaml
```
cat > psa-workload.yaml
```
```
 apiVersion: v1
 kind: Pod
 metadata:
   name: nginx
   labels:
     app: nginx
 spec:
   containers:
   - name: nginx
     image: nginx
     securityContext:
       privileged: true
```
## Apply the manifest to the baseline-ns namespace
```
 kubectl apply -f psa-workload.yaml --namespace=baseline-ns
```
The baseline policy allows the Pod to deploy in the namespace.

## Verify that the Pod deployed successfully
```
 kubectl get pods --namespace=baseline-ns -l=app=nginx
```
## Apply the manifest to the restricted-ns namespace
```
 kubectl apply -f psa-workload.yaml --namespace=restricted-ns
```

## View policy violations in the audit logs
Run query in the Logs Explorer
```
 resource.type="k8s_cluster"
 protoPayload.response.reason="Forbidden"
 protoPayload.resourceName="core/v1/namespaces/restricted-ns/pods/nginx"
```

## Rotate IP Address and Credentials
You perform IP and credential rotation on your cluster. It is a secure practice to do so regularly to reduce credential lifetimes. 
While there are separate commands to rotate the serving IP and credentials, rotating credentials additionally rotates the IP as well.

In Cloud Shell run this command
```
gcloud container clusters update $CLUSTER--zone $ZONE --start-credential-rotation
```
Enter Y to continue

After the command completes in the Cloud Shell the cluster will initiate the process to update each of the nodes. 
That process can take up to 15 minutes for your cluster. 
The process also automatically updates the kubeconfig entry for the current user.

The cluster master now temporarily serves the new IP address in addition to the original address.
You must update the kubeconfig file on any other system that uses kubectl or API to access the master before completing the rotation process to avoid losing access.

## To complete the credential and IP rotation tasks execute the following command
```
gcloud container clusters update $CLUSTER--zone $ZONE --complete-credential-rotation
```
This finalizes the rotation processes and removes the original cluster ip-address.
If the credential rotation fails to complete and returns an error message, run the below command.

```
gcloud container clusters upgrade $CLUSTER--zone $ZONE --node-pool=default-pool
```
Enter Y to continue

After the cluster has successfully upgraded, re-execute the following command
```
gcloud container clusters update $CLUSTER--zone $ZONE --complete-credential-rotation
```
## Delete created resources
```
kubectl delete -f psa-workload.yaml
```
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End


