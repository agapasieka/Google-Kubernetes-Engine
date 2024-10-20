<!-- Overview -->
## Overview
In this lab you will build a Docker container image from provided code and a Dockerfile using Cloud Build. You will then upload the container to the Artifact Registry.
Follow the steps in Cloud Shell or use local terminal of your choice. 

<!-- Task1 -->
## Enable APIs
  ```sh
  gcloud services enable \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
  ```

<!-- Task2 -->
## Building containers with DockerFile and Cloud Build

1. Create quickstart.sh file which will represent an application inside the container
  ```sh
    cat << EOF > quickstart.sh
        #!/bin/sh
        echo "Hello, world! The time is $(date)."
EOF
  ```  

2. Create Dockerfile file and use it as a build configuration script with Cloud Build
  ```sh
    cat <<EOF > Dockerfile
    FROM alpine
    COPY quickstart.sh /
    CMD ["/quickstart.sh"]
EOF
  ```

3. Make the quickstart.sh script executable
  ```sh
  chmod +x quickstart.sh
  ``` 

4. Specify REGION variable and create a new Docker repository named quickstart-docker-repo 
  ```sh
export REGION=
gcloud artifacts repositories create quickstart-docker-repo --repository-format=docker \
    --location=$REGION --description="Docker repository"
  ```

5. Build the Docker container image in Cloud Build
  ```sh
  gcloud builds submit --tag $REGION-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo/quickstart-image:tag1
  ``` 

<!-- Task3 -->
## Building containers with a YAML configuration file and Cloud Build

1. Create file called cloudbuild.yaml
  ```sh
  cat <<EOF > cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
    args: [ 'build', '-t', 'YourRegionHere-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo/quickstart-image:tag1', '.' ]
images:
- 'YourRegionHere-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo/quickstart-image:tag1'
EOF
  ```

2. Insert the region you specified in variable earlier into the yaml file
 ```sh
  sed -i "s/YourRegionHere/$REGION/g" cloudbuild.yaml
 ```   

3. Verify the region was inserted
  ```sh
   cat cloudbuild.yaml
  ```

4. Start a Cloud Build using cloudbuild.yaml as the build configuration file
  ```sh
   gcloud builds submit --config cloudbuild.yaml
  ```

5. Verify two versions of quickstart-image are now present in quickstart-docker-repo
 ```sh
   gcloud artifacts docker images list $REGION-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo
 ```

<!-- Task3 -->
## Building and testing containers with a custom YAML configuration file and Cloud Build

1. Alter the quickstart.sh file
 ```sh
  cat <<EOF > quickstart.sh
    #!/bin/sh
    if [ -z "$1" ]
    then
	     echo "Hello, world! The time is $(date)."
	     exit 0
    else
	     exit 1
    fi
EOF
  ``` 

2. Create a new custom cloud build configuration file called cloudbuild2.yaml. This has been slightly modified to demonstrate Cloud Build's ability to test the containers it has built.
 ```sh
 cat <<EOF > cloudbuild2.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
   args: [ 'build', '-t', 'YourRegionHere-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo/quickstart-image:tag1', '.' ]
- name: 'YourRegionHere-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo/quickstart-image:tag1'
   args: ['fail']
images:
- 'YourRegionHere-docker.pkg.dev/${DEVSHELL_PROJECT_ID}/quickstart-docker-repo/quickstart-image:tag1'
EOF
  ```

3.  Insert your region value into the yaml file
  ```sh
   sed -i "s/YourRegionHere/$REGION/g" cloudbuild2.yaml
  ```

4. Start a Cloud Build using cloudbuild2.yaml
  ```sh
  gcloud builds submit --config cloudbuild2.yaml
  ```

5. The output from the command ends with text like this:

BUILD FAILURE: Build step failure: build step 1 "us-east1-docker.pkg.dev/qwiklabs-gcp-02-1c7ba5c697a0/quickstart-docker-repo/quickstart-image:tag1" failed: starting step container failed: Error response from daemon: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "fail": executable file not found in $PATH: unknown
ERROR: (gcloud.builds.submit) build 96c4a454-be06-4010-aa7c-da57c14165f4 completed with status "FAILURE"


<!-- Task4 -->
## Delete created resources 
  ```sh
gcloud artifacts repositories delete quickstart-docker-repo --location=$REGION
  ```

## The End

