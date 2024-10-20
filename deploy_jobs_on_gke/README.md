<!-- Overview -->
## Overview
In this lab, you learn how to perform the following tasks:

* Define, deploy and clean up a GKE Job
* Define, deploy and clean up a GKE CronJob

In GKE, a **Job** is a controller that handles one-time tasks that eventually finish. It starts a task (usually in a Pod) and keeps track of it until it’s done. Once the task completes successfully, the Job also finishes. Unlike other controllers that keep things running (like Deployments, which always try to keep a certain number of Pods running), a Job is just about making sure the task gets done once.
A **CronJob** is like a Job, but it runs at specific times or intervals (similar to setting an alarm). For example, you can use a CronJob to perform regular tasks like backups, cleaning up old files, or generating reports. It uses Job objects to do the actual work b

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
## Create file called example-job.yaml that contains a sample Job that computes the value of Pi to 2,000 places and then prints the result
  ```sh
cat << EOF > example-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  # Unique key of the Job instance
  name: example-job
spec:
  template:
    metadata:
      name: example-job
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl"]
        args: ["-Mbignum=bpi", "-wle", "print bpi(2000)"]
      # Do not restart containers after they exit
      restartPolicy: Never
EOF
  
  ``` 

<!-- Task5 -->
## Create a Job from this file
  ```sh
kubectl apply -f example-job.yaml
  ```

<!-- Task6 -->
## Check the status of this Job
  ```sh
kubectl describe job example-job  
  ``` 

<!-- Task7 -->
## View all Pod resources in your cluster, including Pods created by the Job which have completed
  ```sh
kubectl get pods
  ```

<!-- Task7 -->
## View all Pod resources in your cluster, including Pods created by the Job which have completed
 ```sh
kubectl get pods  
 ```   

<!-- Task8 -->
## Clean up and delete the Job
  ```sh
kubectl get jobs   
  ```

<!-- Task9 -->
## Retrieve the log file from the Pod that ran the Job
  ```sh
kubectl logs [POD-NAME]   
  ```

<!-- Task10 -->
## Delete the Job
 ```sh
kubectl delete job example-job   
 ```

<!-- Task11 -->
## Define a CronJob manifest called example-cronjob.yaml to deploy a new container every minute that prints the time, date and "Hello, World!".
 ```sh
cat << EOF > example-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo "Hello, World!"
          restartPolicy: OnFailure
EOF 
  ``` 

<!-- Task12 -->
## Create a Job from this file
 ```sh
kubectl apply -f example-cronjob.yaml
  ```

<!-- Task13 -->
##  Get a list of the Jobs in the cluster
  ```sh
 kubectl get jobs  
  ```

<!-- Task14 -->
##  Check the status of this Job
  ```sh
kubectl describe job [job_name]  
  ```

<!-- Task14 -->
##  View the output of the Job by querying the logs for the Pod
  ```sh
kubectl logs [POD-NAME]
  ```

<!-- Task15 -->
##  Delete all jobs
  ```sh
kubectl delete cronjob hello
  ```

<!-- Task15 -->
## Verify that the jobs were deleted
 ```sh
kubectl get jobs
  ```



## The End
