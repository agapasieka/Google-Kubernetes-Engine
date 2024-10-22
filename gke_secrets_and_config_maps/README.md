# Overview

In this lab, you will perform the following tasks:

* Create secrets by using the kubectl command and manifest files.
* Create ConfigMaps by using the kubectl command and manifest files.
* Consume secrets in containers by using environment variables or mounted volumes.
* Consume ConfigMaps in containers by using environment variables or mounted volumes.

Encrypted configuration information is stored as secrets. 
Unencrypted configuration information is stored as ConfigMaps.

This approach avoids hard coding such information into code bases. 
Credentials (like API keys) that belong in secrets should never travel inside code repositories like GitHub (unless they are encrypted before going in, but even then it is better to keep them separate).

# Work with secrets
In this task, you authenticate containers with Google Cloud in order to access Google Cloud services. 
You set up a Cloud Pub/Sub topic and subscription, try to access the Cloud Pub/Sub topic from a container running in GKE, and see that the access request fails.

To properly access the pub/sub topic, you create a service account with credentials, and pass those credentials through Kubernetes Secrets.

## Create service account with no permissions 
```
gcloud iam service-accounts create no-permissions --display-name="no-permissions"
```
## Set the environment variable for the zone and cluster name
```
export ZONE=
export CLUSTER=
export SA=no-permissions@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com
```
## Create a GKE cluster
```
gcloud container clusters create $CLUSTER --num-nodes 2 --zone $ZONE --enable-ip-alias --service-account=$SA
```  

## Configure access to your cluster
```
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```
## Set up Cloud Pub/Sub and deploy an application to read from the topic
```
export pubsub_topic=echo
export pubsub_subscription=echo-read

gcloud pubsub topics create $pubsub_topic
gcloud pubsub subscriptions create $pubsub_subscription --topic=$pubsub_topic
```
## Deploy an application to read from Cloud Pub/Sub topics
Create a deployment with a container that can read from Cloud Pub/Sub topics. 
Since specific permissions are required to subscribe to, and read from, Cloud Pub/Sub topics this container needs to be provided with credentials in order to successfully connect to Cloud Pub/Sub.
```
cat > pubsub.yaml
```
```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: pubsub
  spec:
    selector:
      matchLabels:
        app: pubsub
    template:
      metadata:
        labels:
          app: pubsub
      spec:
        containers:
        - name: subscriber
          image: gcr.io/google-samples/pubsub-sample:v1
```
Ctrl + C to exit

## Deploy the application
```
kubectl apply -f pubsub.yaml
```
## List the Pods
```
kubectl get pods -l app=pubsub
```
The status of Pod reports an error. 

## Inspect the logs from the Pod
```
kubectl logs -l app=pubsub
```
The error message displayed at the end of the log indicates that the application doesn't have permissions to query the Cloud Pub/Sub service.

## Create service account credentials
Create a new service account and grant it access to the pub/sub subscription that the test application is attempting to use. 
Instead of changing the service account of the GKE cluster nodes, you will generate a JSON key for the service account, and then securely pass the JSON key to the Pod via Kubernetes Secrets.
```
gcloud iam service-accounts create pubsub-app --display-name="pubsub-app"

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
  --member="serviceAccount:pubsub-app@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/pubsub.subscriber"

gcloud iam service-accounts keys create ~/credentials.json --iam-account="pubsub-app@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com"

```
## Import credentials as a secret
In Cloud Shell, click the three dots (more icon) in the Cloud Shell toolbar, click Upload and upload the credentials.json file from your local machine to the Cloud Shell VM.

## Save the credentials.json key file to a Kubernetes Secret named pubsub-key
```
kubectl create secret generic pubsub-key --from-file=key.json=$HOME/credentials.json
```
## Configure the application with the secret
Create a new deployment to include the following changes:

* Add a volume to the Pod specification. This volume contains the secret.
* The secrets volume is mounted in the application container.
* The GOOGLE_APPLICATION_CREDENTIALS environment variable is set to point to the key file in the secret volume mount.

```
cat > pubsub-secret.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pubsub
spec:
  selector:
    matchLabels:
      app: pubsub
  template:
    metadata:
      labels:
        app: pubsub
    spec:
      volumes:
      - name: google-cloud-key
        secret:
          secretName: pubsub-key
      containers:
      - name: subscriber
        image: gcr.io/google-samples/pubsub-sample:v1
        volumeMounts:
        - name: google-cloud-key
          mountPath: /var/secrets/google
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /var/secrets/google/key.json
```
Ctrl + C to exit

## Deploy the pubsub-secret.yaml configuration file
```
kubectl apply -f pubsub-secret.yaml
```
## Check the Pod status
```
kubectl get pods -l app=pubsub
```
The Pod status should be Running.

## Test receiving Cloud Pub/Sub messages
Publish a message to the Cloud Pub/Sub topic created earlier
```
gcloud pubsub topics publish $pubsub_topic --message="Hello, world!"
```
Within a few seconds, the message should be picked up by the application and printed to the output stream.

## Inspect the logs from the deployed Pod
```
kubectl logs -l app=pubsub
```

# Working with ConfigMaps
ConfigMaps bind configuration files, command-line arguments, environment variables, port numbers, and other configuration artifacts to your Pods' containers and system components at runtime.

ConfigMaps enable you to separate your configurations from your Pods and components. 
But ConfigMaps aren't encrypted, making them inappropriate for credentials. 
This is the difference between secrets and ConfigMaps: secrets are encrypted and are therefore better suited for confidential or sensitive information such as credentials.

ConfigMaps are better suited for general configuration information such as port numbers.

## Create ConfigMap
```
kubectl create configmap sample --from-literal=message=hello
```
## See how Kubernetes ingested the ConfigMap
```
kubectl describe configmaps sample
```
## Create a ConfigMap from a file called sample2.properties
```
cat > sample2.properties
```
```
message2=world
foo=bar
meaningOfLife=42
```
Ctrl + C to exit

```
kubectl create configmap sample2 --from-file=sample2.properties
```
## See how Kubernetes ingested the ConfigMap
```
kubectl describe configmaps sample2
```

## Use manifest files to create ConfigMaps
Create config-map-3.yaml file that contains a ConfigMap definition called sample3
```
apiVersion: v1
data:
  airspeed: africanOrEuropean
  meme: testAllTheThings
kind: ConfigMap
metadata:
  name: sample3
  namespace: default
  selfLink: /api/v1/namespaces/default/configmaps/sample3
```
Ctrl + C to exit

## Create the ConfigMap from the YAML file
```
kubectl apply -f config-map-3.yaml
```
## See how Kubernetes ingested the ConfigMap
```
kubectl describe configmaps sample3
```

## Use environment variables to consume ConfigMaps in containers
In order to access ConfigMaps from inside Containers using environment variables the Pod definition must be updated to include one or more configMapKeyRefs.
Create CLoud Pub/Sub demo Deployment called pubsub-configmap.yaml that includes setting to import environmental variables from the ConfigMap into the container.

```
cat > pubsub-configmap.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pubsub
spec:
  selector:
    matchLabels:
      app: pubsub
  template:
    metadata:
      labels:
        app: pubsub
    spec:
      volumes:
      - name: google-cloud-key
        secret:
          secretName: pubsub-key
      containers:
      - name: subscriber
        image: gcr.io/google-samples/pubsub-sample:v1
        volumeMounts:
        - name: google-cloud-key
          mountPath: /var/secrets/google
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /var/secrets/google/key.json
        - name: INSIGHTS
          valueFrom:
            configMapKeyRef:
              name: sample3
              key: meme
```
Ctrl + C to exit

## Apply the config file
```
kubectl apply -f pubsub-configmap.yaml
```
Now your application has access to an environment variable called INSIGHTS, which has a value of testAllTheThings.

## Verify that the environment variable has the correct value by gaining shell access to your Pod
```
kubectl get pods
kubectl exec -it [MY-POD-NAME] -- sh
```
## Print a list of environment variables
```
printenv
```
INSIGHTS=testAllTheThings should appear in the list.

## Exit the container's shell session
```
exit
```

## Use mounted volumes to consume ConfigMaps in containers
You can populate a volume with the ConfigMap data instead of (or in addition to) storing it in an environment variable.

In this Deployment the ConfigMap named sample-3 that you created earlier in this task is also added as a volume called config-3 in the Pod spec. 
The config-3 volume is then mounted inside the container on the path /etc/config. 
The original method using Environment variables to import ConfigMaps is also configured.

Create Deployment file called pubsub-configmap2.yaml
```
cat > pubsub-configmap2.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pubsub
spec:
  selector:
    matchLabels:
      app: pubsub
  template:
    metadata:
      labels:
        app: pubsub
    spec:
      volumes:
      - name: google-cloud-key
        secret:
          secretName: pubsub-key
      - name: config-3
        configMap:
          name: sample3
      containers:
      - name: subscriber
        image: gcr.io/google-samples/pubsub-sample:v1
        volumeMounts:
        - name: google-cloud-key
          mountPath: /var/secrets/google
        - name: config-3
          mountPath: /etc/config
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /var/secrets/google/key.json
        - name: INSIGHTS
          valueFrom:
            configMapKeyRef:
              name: sample3
              key: meme
```
Ctrl + C to exit

## Apply the configuration file
```
kubectl apply -f pubsub-configmap2.yaml
```
## Reconnect to the container's shell session to see if the value in the ConfigMap is accessible. 
```
kubectl get pods
kubectl exec -it [MY-POD-NAME] -- sh
```
## Navigate to the appropriate folder
```
cd /etc/config
```
## List the files in the folde
```
ls
```
The output will list: airspeed  meme

## View the contents of one of the files
```
cat airspeed
```
## Exit the container's shell
```
exit
```

## Delete created resources
```
kubectl delete -f pubsub.yaml
kubectl delete -f pubsub-secret.yaml
kubectl delete -f config-map-3.yaml
kubectl delete -f pubsub-configmap.yaml
kubectl delete -f pubsub-configmap2.yaml
```
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End
