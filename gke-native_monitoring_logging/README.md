# Overview

In this lab, you will perform the following tasks:

* Use Kubernetes Engine Monitoring to view cluster and workload metrics
* Use Cloud Monitoring Alerting to receive notifications about the cluster’s health

## Using Kubernetes Engine Monitoring
Google Kubernetes Engine includes managed support for Monitoring.
Create a new cluster with Kubernetes Engine Monitoring support and then perform typical monitoring tasks using the Kubernetes Engine monitoring and logging interface.

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
Click the cluster name to view the cluster details and under the Features heading, you can see the Logging and Cloud Monitoring settings that set the logging type to System, Workloads and System.


## Deploy a sample workload to your GKE cluster
This workload consists of a deployment of three pods running a simple Hello World demo application.
```
cat > hello-v2.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-v2
spec:
  replicas: 3
  selector:
    matchLabels:
      run: hello-v2
  template:
    metadata:
      labels:
        run: hello-v2
        name: hello-v2
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:2.0
        name: hello-v2
        ports:
        - containerPort: 8080
          protocol: TCP
```
Ctrl + C to exit

## Deploy the application
```
kubectl create -f hello-v2.yaml
```
## Verify the deployment
```
kubectl get deployments
```

## Deploy the GCP-GKE-Monitor-Test application
Deploy the GCP-GKE-Monitor-Test application to the default namespace of your GKE cluster. This workload has a deployment consisting of a single pod that is then exposed to the internet via a LoadBalancer service.
```
export PROJECT_ID="$(gcloud config get-value project -q)"
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
ln -s ~/training-data-analyst/courses/ak8s/v1.1 ~/ak8s
cd Monitoring/gcp-gke-monitor-test
gcloud builds submit --tag=gcr.io/$PROJECT_ID/gcp-gke-monitor-test .
cd ..
```
## Replace a placeholder value in the gcp-gke-monitor-test.yaml file with the Docker image you just pushed to gcr.io
```
sed -i "s/\[DOCKER-IMAGE\]/gcr\.io\/${PROJECT_ID}\/gcp-gke-monitor-test\:latest/" gcp-gke-monitor-test.yaml
```
## Deploy the application
```
kubectl create -f gcp-gke-monitor-test.yaml
```
## Verify the deployment
```
kubectl get deployments
```
## Verify the service
```
kubectl get service
```
# Using the GCP-GKE-Monitor-Test application
Use the GCP-GKE-Monitor-Test application to explore different aspects of Kubernetes Engine Monitoring. The tool is composed of four sections:

* Generate CPU Load
* Custom Metrics
* Log Test
* Crash the Pod

Open your web browser and navigate to the EXTERNAL-IP address of the service
## Get the service IP
```
kubectl get service
```
In the Generate CPU Load section, click the Start CPU Load button. Note that the STATUS text will change when the load generator starts running.

## Start collecting custom metrics
Start a process within the GCP-GKE-Monitor-Test tool which creates a Custom Metric Descriptor within Cloud Monitoring. Later, when the tool begins sending the custom metric data, Monitoring will associate the data with this metric descriptor. Note that Monitoring can often automatically create the custom metric descriptors for you when you send the custom metric data, but creating the descriptor manually gives you more control over the text that appears in the Monitoring interface, making it easier for you to find your data in the Metric Explorer.

1. In the GCP-GKE-Monitor-Test tool, in the Custom Metrics section, click the Start Monitoring button.

2. Click Increase Users Counter and repeat until the Current User Count is set to 10 users.

## Generate test log messages
Use the GCP-GKE-Monitor-Test tool to create sample text-based logs which you will later view in Cloud Monitoring.

1. In the GCP-GKE-Monitor-Test tool, in the Log Test section, click the Enable Debug Logging button to increase the number of logs the tool generates.
2. Click the other four log entry buttons to generate some additional sample log messages. It's important to select a variety of severity levels so that you can see how the different message types are displayed in Monitoring.

# Using Kubernetes Engine Monitoring
Use Kubernetes Engine Monitoring to view the current health of your GKE cluster and the two workloads running on it.

Setup a Monitoring workspace that's tied to your Google Cloud Project. 
1. In the Cloud Console, click on Navigation menu > Monitoring.
2. Wait for your workspace to be provisioned.

Examine each section in the interface:
* The Clusters, Nodes, and Pods sections allow you to check the health of particular elements in the cluster. You can also use this to inspect the pods which are running on a particular node in the cluster.

* To see the details of your cluster, click on the cluster element.

* The Workloads section is very helpful, especially when looking for workloads which do not have services exposed.

* The Kubernetes services section organizes the services configured in your environment by cluster, then by namespace (the administrative barrier or partition within the cluster), and then shows the various services available to users within that namespace. You can see more details on each service by clicking on their name.

* The Namespaces section shows the list of namespaces within the cluster.

# Creating alerts with Kubernetes Engine Monitoring
Create an Alert Policy
1. In the Cloud Console, from the Navigation menu, select Monitoring > Detect >Alerting.
2. Click + Create Policy.
3. Click on Select a metric dropdown.
4. Uncheck the Active option.
5. Type Kubernetes Container in filter by resource and metric name.
6. Click on Kubernetes Container > Container.
7. Select CPU request utilization.
8. Click Apply.
9. Set Rolling windows to 1 min.
10. Click Next.
11. Set Threshold position to Above Threshold.
12. Set 0.99 as your Threshold value.
13. Click Next.
14. Configure Notification Channel
15. Name the alert CPU request utilization
16. Review the alert and click Create Policy.
