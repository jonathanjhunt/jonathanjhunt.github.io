---
layout: post
title: "Building a 3-tier application to demonstrate Kubernetes Pod Autoscaling"
subtitle: "React, Express, Mongo, K8s, Prometheus, Grafana, K6"
date: 2024-07-05 13:45:13 -0000
description: "React, Express, Mongo, K8s, Prometheus, Grafana, K6"
categories: [Guides]
tags: [DevOps, SRE, kubernetes, React, Express, Mongo, Prometheus, Grafana, K6]
pin: false
math: true
comments: true
mermaid: true
image:
  path: /assets/img/posts/kubernetes.png
  lqip: /assets/img/posts/kubernetes_sqip.svg
  alt: Kubernetes - An Orchestration Tool for Containerised Applications
---


# Three Tier Web App - K8s | React | Express | Prometheus | Grafana | K6


## Effortless Scaling: Mastering Kubernetes, React, and Express for Web Apps


In the ever-evolving landscape of web applications, ensuring robust performance and scalability is crucial. Kubernetes has emerged as a powerful orchestration tool, revolutionizing the way developers deploy, manage, and scale containerized applications. Its ability to automatically scale in and out based on load and demand makes it an essential technology for modern web applications.

Kubernetes, often abbreviated as K8s, offers dynamic scaling capabilities that respond to real-time traffic patterns. This ensures that applications maintain optimal performance during peak loads and efficiently utilize resources during quieter periods. The Horizontal Pod Autoscaler (HPA) in Kubernetes plays a pivotal role in this process by adjusting the number of pod replicas according to the observed CPU utilization or other select metrics.

On the frontend, React is a highly efficient library for building user interfaces. Its component-based architecture promotes reusable code, making development faster and more maintainable. React's virtual DOM optimizes rendering performance, providing a smooth user experience even in complex applications.

For the backend, Express.js is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications. It facilitates the rapid development of server-side logic and APIs, integrating seamlessly with databases like MongoDB to handle data operations.

Together, React and Express form a powerful duo for building scalable, efficient, and high-performance web applications. This guide will walk you through setting up a three-tier web application using these technologies, deployed on a Kubernetes cluster. You'll experience firsthand the autoscaling capabilities of Kubernetes, ensuring your application can handle varying loads with ease.

## Monitoring with Prometheus and Grafana

Effective monitoring is crucial in maintaining the health and performance of applications running in Kubernetes. Prometheus and Grafana are an exemplary combination for this purpose.

Prometheus is a powerful monitoring system and time-series database that is well-suited for Kubernetes environments. It collects metrics from various sources and stores them efficiently, allowing for powerful querying and alerting capabilities. Prometheus integrates seamlessly with Kubernetes, automatically discovering and monitoring services and pods. This allows developers to gain deep insights into application performance, resource usage, and potential issues.

Grafana complements Prometheus by providing a robust platform for visualizing and analyzing metrics. With its rich set of features and customizable dashboards, Grafana makes it easy to interpret complex data and trends. This visualization capability is invaluable for diagnosing problems, optimizing performance, and ensuring that the application meets its service level objectives.

Together, Prometheus and Grafana form a comprehensive monitoring solution that helps ensure the reliability and efficiency of applications deployed on Kubernetes.

## Load Testing with k6

Ensuring that an application can handle expected load and performance requirements is a critical aspect of development. k6 is an open-source load testing tool that is both powerful and easy to use, making it an excellent choice for testing web applications.

k6 allows developers to write load tests in JavaScript, making it accessible and straightforward to integrate into the development workflow. It simulates real-world traffic conditions, allowing you to test how your application performs under various loads. This helps identify potential bottlenecks and ensures that the application can scale appropriately to meet user demand.

By incorporating k6 into your testing strategy, you can:

1. Verify the performance and reliability of your application under load.
2. Identify and resolve performance bottlenecks before they impact users.
3. Ensure that your Kubernetes autoscaling policies are effective and responsive.

Load testing with k6 is essential for delivering a robust and scalable application that can handle real-world usage scenarios.

## Get Started

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> The repo containing the application code and start script can be found at:
[jonathanjhunt/3-tier-app-with-k8s](https://github.com/jonathanjhunt/3-tier-app-with-k8s)
{: .prompt-info }

<!-- markdownlint-restore -->

1. Build the app with one install/run script.
2. Run load tests using [k6](https://k6.io)
3. View the cluster performance metrics in [Grafana](https://grafana.com)
4. Experience the power of K8s autoscaling in action!


![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/e161b058-507b-42d0-9964-7f42a066270f)


## Prerequisites

| Service     | Requirement                                                                                                      | Link                                                                                                                   |
|-------------|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Minikube    | Minikube allows you to run a Kubernetes cluster using Docker containers inside your local environment.           | [Minikube Documentation](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download)    |
| Helm        | Helm is a package manager for Kubernetes environments. It is used in this installation to install Prometheus & Grafana, the monitoring tool of choice. | [Helm Documentation](https://helm.sh/docs/intro/install/)                                                             |
| Docker      | Docker is used to run the underlying containers that Minikube is launched on. Without the Docker engine, Minikube will be unable to start. You can choose to install Docker Desktop, or just the Docker Engine. | [Docker Documentation](https://docs.docker.com/engine/install/)                                                       |
| K6          | K6 is a load-testing tool designed for Kubernetes, built by Grafana Labs. It is used in this application to simulate load on the pods. | [K6 Documentation](https://grafana.com/oss/k6/?src=k6io)                                                               |


## Setup & Instructions

### 1. Check Pre-requisites
Ensure that all the pre-requisites are installed. Links to download instructions are provided above. 
### 2. Clone this Repo
Clone this repo using the command:

    git clone https://github.com/jonathanjhunt/euros-vote-app

### 3. Start the application
Navigate to the folder of the cloned repo and run the command:

    sh start_app.sh

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/c7f212a2-5d4e-47b2-a5d6-a129877426e9)


The startup script will:

1. Check that pre-requisites are installed
2. Spin up the minikube cluster
3. Apply the kubernetes manifests
4. Seed the mongoDB database
5. Install the Prometheus-Grafana stack with helm
6. Expose the Frontend and Grafana Service so they are accessible via localhost

### 4. Check the UI

Once the script has completed, the application should be accessible at:
**Front-End**: [http://localhost:30000](http://localhost:30000)

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/6aa5855c-00a2-4527-98f0-edf7e09facd7)


### 5. Access Grafana and Configure the Dashboard

Grafana should be accessible via:
**Grafana**: [http://localhost:30001](http://localhost:30001)

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/07b62ac7-e451-4f6c-9786-220f48f7a422)

| Username | Password|
|--|--|
|**admin**  | **prom-operator** |



Once you have logged in:
1. Select Dashboards on the left panel
2. Click the blue 'New' button
3. Select Import

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/42ea9819-8358-4d48-a342-34f568859baf)



4. In the cloned github repo, copy the contents from the JSON file found under 
		`/monitoring/grafana/euros-dashboard.json`
5. Paste in the box under *Import via dashboard JSON model*
6. Select Load
7. Select Import
### Run K6 Load Test

To simulate load through the kubernetes cluster, I have used K6. K6 is a tool built by grafana-labs that can create virtual users and send http requests to endpoints in order to simulate real load on the system. This can help demonstrate the autoscaling capabilities that have been configured within this kubernetes app. 

Before beginning the load test, take a look at the number of pods running on the system, as well as the average cpu load per pod. 

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/5948009e-bc14-40fb-916f-04f30510b8b4)


To begin the load test, make sure K6 is installed, and then run:

    k6 run load-testing/k6/full-ramp-load-test.js

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/e41dbb42-4142-4075-8d3e-5f5f93cddfad)


The load test will begin, ramping up the number of virtual users. This mocks real activity on the web-app and gives the system time to scale in and out as required. 

As the number of VU's increases, check Grafana to see the average CPU of the pods and the number of Pods within the deployment. 

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/8bd83b2c-05ba-49c2-b9fd-b4a8cbd9df55)


Thats it! Kubernetes Pods within the cluster scale up and down to accomodate for additional load on the system. 

### Known Issues / Caveats

- The front end pods are exposed using port forwarding with service types **nodePort**. This is due to known issues with minikube tunnel preventing connectivity to the service type **Load Balancer**. This means that although the frontend pods can scale, all network traffic goes through to only one pod. This puts a bottle neck on the amount of load that can be applied to the system. Additionally for this reason, it is not advised that this code is deployed to a production environment. **This code is only designed to be run locally or in a development environment with minikube**

### Terminating the Application and Cluster
To shut down the application and the minikube cluster, simply run:

    minikube delete

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/436ae1f7-022b-4e89-a493-051bf654408f)

## Architecture

<img width="486" alt="image" src="https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/ef0cef96-41fc-4d6e-ae71-2013685fad07">

### Kubernetes Configuration
The Kubernetes configuration for this application has been seperated into 3 seperate files for each tier of the application:

#### Deployment Definitions

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  replicas: 2
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
        - name: frontend
          image: jonathanjhunt/euros-vote-app-frontend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "150Mi"
              cpu: "200m"
            limits:
              memory: "250Mi"
              cpu: "200m"

...
```


The deployment definition files for each tier of the application follow a similiar template. The deployment defines the application tiers information, such as image used, nmber of pods, tags and resource requirements/requests.

| key | definition |
|--|--|
| matchLabels | Defines the tags that the service yaml will use to know what pods to attatch to |
| replicas | base number of pods for the deployment  |
| image | the image that it pulls from dockerhub to run on the pod |
| imagePullPolicy | define wether you want to always pull the latest image when building a new pod |
| containerPort | the port that the application exposes itself on |
| resources:requests | the minimum hardware resources that are assigned to a pod |
| resources:limits | the maximum hardware resources that a pod can consume |

#### Service Definitions

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
    tier: frontend
  ports:
  - protocol: "TCP"
    port: 80
    targetPort: 80
    nodePort: 30000
  type: NodePort
...
```


The service definition file acts as the network connection between pods. Services are attached to pods and allows connectivity between different applications, as well as exposing connectivity to the internet. 

| key | definitions |
|--|--|
|selector/app  | Defines tags that pods must match to attach to this service |
| selector/tier | Defines tags that pods must match to attach to this service |
| port | Port that the application exposes itself on |
| nodePort  | The port that kubernetes exposes when using type nodePort |
| type | Define the type of load balancer, ClusterIP (pod to pod communication), LoadBalancer (preferred approach for external connection, doesnt work well with minikube), nodePort (Easiest method for exposing in local development)|


#### HPA Definition

The HPA (horizontal pod autoscaling) definition file configurs the pods to autoscale based on the average load on a pod. This allows the three tier-application to scale in and out based on usage. 

```yaml
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 30
...
```

| key | definition |
|--|--|
| scaleTargetRef:name  | specifies which pods to attach the scale policy to |
| minReplicas | minimum number of pods for the application labelled "frontend"  |
| maxReplicas  | maximum number of pods for the application labelled "frontend"   |
| targetCPUUtilizationPercentage | the target CPU for pods to remain at. |

