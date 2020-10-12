### Hej Hej! Welcome to my project Catch-n-replace 👋

#### This project aims to demonstrate the usage of RegEx in a nodejs webapp and deployment of the application into various cloud deployment strategies. In this project we will be replacing a particular string with another string using Regular Expression.


🌱 All Set, let's Go .... 👯

### Prequisites

1. Exposure to ExpressJS, NodeJS and Javascript Concepts
2. Node 12 or above
3. Docker 
4. Google Cloud Platform Account 

### Building the App
A. On Local host
1. Clone the repository and head to the root directory of the folder

cd /catch-n-replace/

2. Install Nodemon or run node app.js on the terminal and head to the browser http://localhost:3211

B. On Container using Docker.

1. Build an Docker image with the following command

docker build -t <docker-image name>:<with tag> <location of the Dockerfile>

in this case 

docker build -t catch-n-replace:latest . (. specifies current location)

### Deploying the app

A. On a VM (Ubuntu or Centos)

1. Install Apache server using "https://ubuntu.com/tutorials/install-and-configure-apache#1-overview" 
2. Build the app with above the mentioned steps.

B. On a GKE

Login to GCP cloud shell and start executing the below commands

#### Clone the repository using

git clone https://github.com/restlessankyyy/catch-n-replace.git and head to the root directory of the project.

#### Pushing the container image into Google Container Registry

Run this command, replacing PROJECT_ID with your Project ID, found in the Console or the Connection Details section of the lab.

gcloud auth configure-docker

docker push gcr.io/PROJECT_ID/catch-n-replace:v1

The initial push may take a few minutes to complete. You'll see the progress bars as it builds  

#### Create a GKE Cluster 

Now you're ready to create your Kubernetes Engine cluster. A cluster consists of a Kubernetes master API server hosted by Google and a set of worker nodes. The worker nodes are Compute Engine virtual machines.

Make sure you have set your project using gcloud (replace PROJECT_ID with your Project ID, found in the console:

gcloud config set project PROJECT_ID 

Create a cluster with two n1-standard-1 nodes (this will take a few minutes to complete):

gcloud container clusters create catch-n-replace \
                --num-nodes 2 \
                --machine-type n1-standard-1 \
                --zone us-central1-a

You can safely ignore warnings that come up when the cluster builds.

The console output should look like this:

Creating cluster hello-world...done.
Created [https://container.googleapis.com/v1/projects/PROJECT_ID/zones/us-central1-a/clusters/catch-n-replace].
kubeconfig entry generated for catch-n-replace.
NAME         ZONE           MASTER_VERSION  MASTER_IP       MACHINE_TYPE   STATUS
catch-n-replace us-central1-a  1.5.7           146.148.46.124  n1-standard-1  RUNNING


Note: You can also create this cluster through the Console by opening the Navigation menu and selecting Kubernetes Engine > Kubernetes clusters > Create cluster


#### Create your pod

A Kubernetes pod is a group of containers tied together for administration and networking purposes. It can contain single or multiple containers. Here you'll use one container built with your Node.js image stored in your private container registry. It will serve content on port 3211.

Create a pod with the kubectl run command (replace PROJECT_ID with your Project ID, found in the console):

kubectl create deployment catch-n-replace \
    --image=gcr.io/PROJECT_ID/catch-n-replace:v1


(Output)

deployment.apps/catch-n-replace created

As you can see, you've created a deployment object. Deployments are the recommended way to create and scale pods. Here, a new deployment manages a single pod replica running the catch-n-replace:v1 image.

To view the deployment, run:

kubectl get deployments

(Output)

NAME         READY   UP-TO-DATE   AVAILABLE   AGE
catch-n-replace   1/1     1            1           1m36s


To view the pod created by the deployment, run:

kubectl get pods

(Output)

NAME                         READY     STATUS    RESTARTS   AGE
catch-n-replace-714049816-ztzrb   1/1       Running   0          6m



Now is a good time to go through some interesting kubectl commands. None of these will change the state of the cluster, full documentation is available here: https://cloud.google.com/container-engine/docs/kubectl/

kubectl cluster-info

kubectl config view

And for troubleshooting :

kubectl get events

kubectl logs <pod-name>

You now need to make your pod accessible to the outside world.

#### Allow external traffic

By default, the pod is only accessible by its internal IP within the cluster. In order to make the catch-n-replace container accessible from outside the Kubernetes virtual network, you have to expose the pod as a Kubernetes service.

From Cloud Shell you can expose the pod to the public internet with the kubectl expose command combined with the --type="LoadBalancer" flag. This flag is required for the creation of an externally accessible IP:

kubectl expose deployment catch-n-replace --type="LoadBalancer" --port=3211

(Output)

service/catch-n-replace exposed
The flag used in this command specifies that are using the load-balancer provided by the underlying infrastructure (in this case the Compute Engine load balancer). Note that you expose the deployment, and not the pod, directly. This will cause the resulting service to load balance traffic across all pods managed by the deployment (in this case only 1 pod, but you will add more replicas later).

The Kubernetes master creates the load balancer and related Compute Engine forwarding rules, target pools, and firewall rules to make the service fully accessible from outside of Google Cloud.

To find the publicly-accessible IP address of the service, request kubectl to list all the cluster services:

kubectl get services

This is the output you should see:

NAME         CLUSTER-IP     EXTERNAL-IP      PORT(S)    AGE
catch-n-replace   10.3.250.149   104.154.90.147   3211/TCP   1m
kubernetes   10.3.240.1     <none>           443/TCP    5m
There are 2 IP addresses listed for your catch-n-replace service, both serving port 3211. The CLUSTER-IP is the internal IP that is only visible inside your cloud virtual network; the EXTERNAL-IP is the external load-balanced IP.

Note: The EXTERNAL-IP may take several minutes to become available and visible. If the EXTERNAL-IP is missing, wait a few minutes and run the command again.

You should now be able to reach the service by pointing your browser to this address: http://<EXTERNAL_IP>:3211

At this point you've gained several features from moving to containers and Kubernetes - you do not need to specify on which host to run your workload and you also benefit from service monitoring and restart. Now see what else can be gained from your new Kubernetes infrastructure.

#### Scale up your service

One of the powerful features offered by Kubernetes is how easy it is to scale your application. Suppose you suddenly need more capacity. You can tell the replication controller to manage a new number of replicas for your pod:

kubectl scale deployment catch-n-replace --replicas=4

(Output)

deployment.extensions/catch-n-replace scaled
You can request a description of the updated deployment:

kubectl get deployment

(Output)

NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
catch-n-replace   4         4         4            4           16m
You may need to run the above command until you see all 4 replicas created. You can also list the all pods:

kubectl get pods

This is the output you should see:

NAME                         READY     STATUS    RESTARTS   AGE
catch-n-replace-714049816-g4azy   1/1       Running   0          1m
catch-n-replace-714049816-rk0u6   1/1       Running   0          1m
catch-n-replace-714049816-sh812   1/1       Running   0          1m
catch-n-replace-714049816-ztzrb   1/1       Running   0          16m
A declarative approach is being used here. Rather than starting or stopping new instances, you declare how many instances should be running at all times. Kubernetes reconciliation loops makes sure that reality matches what you requested and takes action if needed.


### Hurray You have successfully deployed your application on GKE and is accessible at http://34.69.244.71:3211/ from anywhere














