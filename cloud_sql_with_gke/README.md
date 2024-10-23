# Overview

Set up a Kubernetes Deployment of WordPress connected to Cloud SQL via the SQL Proxy. 
The SQL Proxy lets you interact with a Cloud SQL instance as if it were installed locally (localhost:3306), and even though you are on an unsecured port locally, the SQL Proxy makes sure you are secure over the wire to your Cloud SQL Instance.

Components to setup:
* GKE cluster
* Cloud SQL Instance to connect to
* Service Account to provide permission for your Pods to access the Cloud SQL Instance, this will be authenticated using Workload Identity
* WordPress on GKE cluster, with the SQL Proxy as a Sidecar, connected to Cloud SQL Instance

In this lab, you learn how to perform the following tasks:

* Create a Cloud SQL instance and database for Wordpress.
* Create credentials and Kubernetes Secrets for application authentication.
* Configure Workload Identity.
* Configure a Deployment with a Wordpress image to use SQL Proxy.
* Install SQL Proxy as a sidecar container and use it to provide SSL access to a CloudSQL instance external to the GKE Cluster.

## Set the environment variable for the zone and cluster name
```
export REGION=
export ZONE=
export CLUSTER=
export Project_ID=$(gcloud config get-value project)
```
## Create a GKE cluster
```
gcloud container clusters create $CLUSTER --num-nodes 2 --zone $ZONE --enable-ip-alias
```
## Configure access to your cluster
```
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```
## Enable Cloud SQL APIs
```
gcloud services enable sqladmin.googleapis.com
```
## Create a Cloud SQL instance
```
gcloud sql instances create sql-instance --tier=db-n1-standard-2 --region=$REGION
```
## Add User account
```  
gcloud sql users create sqluser --instance=$INSTANCE_NAME --password=sqlpassword --host='%'
```
## Create an environment variable to hold your Cloud SQL instance connection name
```
export SQL_NAME=
```
## Connect to your Cloud SQL instance
```
gcloud sql connect sql-instance
```
When prompted to enter the root password press enter. The root SQL user password is blank by default.
The mysql> prompt appears indicating that you are now connected to the Cloud SQL instance using the MySQL client.

## Create the database required for Wordpress
```
create database wordpress;
```
## Select the wordpress database
```
use wordpress;
```
## List tables
```
show tables;
```
This will report Empty set as you have not created any tables yet.

## Exit the MySQL client
```
exit;
```

## Prepare a Service Account with permission to access Cloud SQL
```
gcloud iam service-accounts create sql-access --display-name="sql-access"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:sql-access@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

gcloud iam service-accounts keys create ~/credentials.json --iam-account="sql-access@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com"
```

## Create Kubernetes Service Account and configure Workload Identity
Create the Kubernetes Service Account
```
kubectl create serviceaccount gkesqlsa
```
## Bind the Google Cloud service account with the Kubernetes Service Account
```
gcloud iam service-accounts add-iam-policy-binding \
--role="roles/iam.workloadIdentityUser" \
--member="serviceAccount:$PROJECT_ID.svc.id.goog[default/gkesqlsa]" \
sql-access@$PROJECT_ID.iam.gserviceaccount.com
```
## Annotate the Kubernetes Service Account with the details of the Google Cloud service account
```
kubectl annotate serviceaccount \
gkesqlsa \
iam.gke.io/gcp-service-account=sql-access@$PROJECT_ID.iam.gserviceaccount.com
```

## Create Secrets
Create two Kubernetes Secrets: one to provide the MySQL credentials and one to provide the Google credentials (the service account).

Create a Secret for your MySQL credentials
```
kubectl create secret generic sql-credentials \
   --from-literal=username=sqluser\
   --from-literal=password=sqlpassword
```
In the Cloud Shell, click More and Select Upload, leave File selected and click on Choose Files and Upload the credentials.json credential file downloaded earlier

## Create a Secret for your Google Cloud Service Account credentials
```
kubectl create secret generic google-credentials\
   --from-file=key.json=credentials.json
```
## Deploy the SQL Proxy agent as a sidecar container
Create a deployment manifest file called sql-proxy.yaml that deploys a demo Wordpress application container with the SQL Proxy agent as a sidecar container.

In the Wordpress container environment settings the WORDPRESS_DB_HOST is specified using the localhost IP address. 
The cloudsql-proxy sidecar container is configured to point to the Cloud SQL instance you created in the previous task. 
The database username and password are passed to the Wordpress container as secret keys, and Workload Identity is configured. 
A Service is also created to allow you to connect to the Wordpress instance from the internet.

## Create sql-proxy.yaml manifest
```
cat > sql-proxy.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      serviceAccountName: gkesqlsa
      containers:
        - name: web
          image: gcr.io/cloud-marketplace/google/wordpress:6.1
          #image: wordpress:5.9
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: 127.0.0.1:3306
            # These secrets are required to start the pod.
            # [START cloudsql_secrets]
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: sql-credentials
                  key: username
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: sql-credentials
                  key: password
            # [END cloudsql_secrets]
        # Change '<INSTANCE_CONNECTION_NAME>' here to include your Google Cloud
        # project, the region of your Cloud SQL instance and the name
        # of your Cloud SQL instance. The format is
        # $PROJECT:$REGION:$INSTANCE
        # [START proxy_container]
        - name: cloudsql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0
          args: 
           - "--structured-logs"
           - "--port=3306"
           -  "<INSTANCE_CONNECTION_NAME>" 
          securityContext:
            runAsNonRoot: true 
---
apiVersion: "v1"
kind: "Service"
metadata:
  name: "wordpress-service"
  namespace: "default"
  labels:
    app: "wordpress"
spec:
  ports:
  - protocol: "TCP"
    port: 80
  selector:
    app: "wordpress"
  type: "LoadBalancer"
  loadBalancerIP: ""
```
Ctrl + C to exit

Important sections:
* In the spec section the Kubernetes Service Account is configured.
* In the Wordpress env section the variable WORDPRESS_DB_HOST is set to 127.0.0.1:3306. This will connect to a container in the same Pod listening on port 3306. This is the port that the SQL-Proxy listens on by default.
* In the Wordpress env section the variables WORDPRESS_DB_USER and WORDPRESS_DB_PASSWORD are set using values stored in the sql-credential Secret you created in the last task.
* In the cloudsql-proxy container section the command switch that defines the SQL Connection name, "INSTANCE_CONNECTION_NAME> contains a placeholder variable that is not configured using a ConfigMap or Secret and so must be updated directly in this example manifest to point to your Cloud SQL instance.
* The Service section at the end creates an external LoadBalancer called "wordpress-service" that allows the application to be accessed from external internet addresses.

## Use sed to update the placeholder variable for the SQL Connection name to the instance name of your Cloud SQL instance
```
sed -i 's/<INSTANCE_CONNECTION_NAME>/'"${SQL_NAME}"'/g'\
   sql-proxy.yaml
```
## Deploy the application
```
kubectl apply -f sql-proxy.yaml
```
## Query the status of the Deployment
```
kubectl get deployment wordpress
```
## List the services in your GKE cluster
```
kubectl get services
```
The external LoadBalancer ip-address for the wordpress-service is the address you use to connect to your Wordpress blog

##  Connect to your Wordpress instance
1. Open a new browser tab and connect to your Wordpress site using the external LoadBalancer ip-address. This will start the initial Wordpress installation wizard.
2. Select English (United States) and click Continue.
3. Enter a sample name for the Site Title.
4. Enter a Username and Password to administer the site.
5. Enter an email address.
6. Click Install Wordpress.

After a few seconds you will see the Success! Notification. You can log in if you wish to explore the Wordpress admin interface but it is not required for the lab.
The initialization process has created new database tables and data in the wordpress database on your Cloud SQL instance. You will now validate that these new database tables have been created using the SQL proxy container.

7. Switch back to the Cloud Shell and connect to your Cloud SQL instance
   ```
   gcloud sql connect sql-instance
   ```
8. Select the wordpress database
   ```
   use wordpress;
   ```
9. Show a number of new database tables that were created when Wordpress was initialized
   ```
   show tables; 
   ```
10. Exit the MySQL client
   ```
   exit
   ```
## Delete created resources
```
gcloud sql instances delete sql-instance --region=$REGION
gcloud iam service-accounts delete sql-access
kubectl delete serviceaccount gkesqlsa
kubectl delete -f sql-proxy.yaml
```
## Delete cluster
```
gcloud container clusters delete $CLUSTER --zone $ZONE
```

# The End

