# Deploy an Application to Google Kubernetes Engine

## Overview
Google Kubernetes Engine (GKE) provides a managed environment for deploying, managing, and scaling your containerized applications using Google infrastructure. The GKE environment consists of multiple machines (specifically Compute Engine instances) grouped to form a container cluster.

This lab provides hands-on practice of container creation and application deployment with GKE.

### Objectives
In this lab, you learn how to perform the following tasks:
* Create a GKE cluster.
* Deploy an application to the cluster.
* Create a Kubernetes service.
* Delete the cluster.

---

## Cluster Orchestration with Google Kubernetes Engine
Google Kubernetes Engine (GKE) clusters are powered by the Kubernetes open-source cluster management system. Kubernetes provides the mechanisms through which you interact with your container cluster. You use Kubernetes commands and resources to deploy and manage applications, perform administrative tasks, set policies, and monitor the health of your deployed workloads.

Kubernetes uses the same design principles that power popular Google services. It provides identical benefits, including automatic management, monitoring, and liveness probes for application containers, automatic scaling, rolling updates, and more. When you run your applications on a container cluster, you're using technology based on Google's 10+ years of experience with running production workloads in containers.

### Kubernetes on Google Cloud
When you run a GKE cluster, you also gain the benefit of advanced cluster management features that Google Cloud provides. These include:
* **Load balancing** for Compute Engine instances
* **Node pools** to designate subsets of nodes within a cluster for additional flexibility
* **Automatic scaling** of your cluster's node instance count
* **Automatic upgrades** for your cluster's node software
* **Node auto-repair** to maintain node health and availability
* **Logging and monitoring** with Cloud Monitoring for visibility into your cluster

---



## Task 1. Set a Default Compute Zone
Your compute zone is an approximate regional location in which your clusters and their resources live. For example, `us-central1-a` is a zone in the `us-central1` region.

In your Cloud Shell session, run the following commands.

1. Set the default compute region:
   ```bash
   gcloud config set compute/region us-east1
   ```
   **Expected output:**
   ```text
   Updated property [compute/region].
   ```

2. Set the default compute zone:
   ```bash
   gcloud config set compute/zone us-east1-c
   ```
   **Expected output:**
   ```text
   Updated property [compute/zone].
   ```

---

## Task 2. Create a GKE Cluster
A cluster consists of at least one cluster master machine and multiple worker machines called nodes. Nodes are Compute Engine virtual machine (VM) instances that run the Kubernetes processes necessary to make them part of the cluster.

> **Note:** Cluster names must start with a letter and end with an alphanumeric, and cannot be longer than 40 characters.

1. Run the following command to create your cluster:
   ```bash
   gcloud container clusters create --machine-type=e2-medium --zone=us-east1-c lab-cluster
   ```
   *You can ignore any warnings in the output. It might take several minutes to finish creating the cluster.*

   **Expected output:**
   ```text
   NAME: lab-cluster
   LOCATION: us-east1-c
   MASTER_VERSION: 1.22.8-gke.202
   MASTER_IP: 34.67.240.12
   MACHINE_TYPE: e2-medium
   NODE_VERSION: 1.22.8-gke.202
   NUM_NODES: 3
   STATUS: RUNNING
   ```

2. Click **Check my progress** to verify the objective.

---

## Task 3. Get Authentication Credentials for the Cluster
After creating your cluster, you need authentication credentials to interact with it.

1. Authenticate with the cluster:
   ```bash
   gcloud container clusters get-credentials lab-cluster
   ```
**Expected output:**
```text
Fetching cluster endpoint and auth data.
kubeconfig entry generated for lab-cluster.
```

---

## Task 4. Deploy an Application to the Cluster
Now that you have authenticated your environment with the GKE cluster, you can deploy a containerized application workload. For this lab, you will run a sample web application called `hello-app`.

1. Run the `kubectl create deployment` command to spin up a new deployment named `hello-server` from the specified container image:
   ```bash
   kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
   ```
   **Expected output:**
   ```text
   deployment.apps/hello-server created
   ```

2. Expose the deployment to the public internet by creating a Kubernetes Service with a LoadBalancer:
   ```bash
   kubectl expose deployment hello-server --type=LoadBalancer --port 8080
   ```
   **Expected output:**
   ```text
   service/hello-server exposed
   ```

3. Inspect the newly created service to find its publicly accessible endpoint:
   ```bash
   kubectl get service
   ```
   **Expected output:**
   ```text
   NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
   kubernetes     ClusterIP      10.3.240.1      <none>          443/TCP          5m
   hello-server   LoadBalancer   10.3.245.137    35.232.115.42   8080:31256/TCP   30s
   ```
   *(Note: The external IP generation takes a few moments. If the `EXTERNAL-IP` field reads `<pending>`, rerun the command after a few seconds).*

4. Open a new browser tab and navigate to the application endpoint using the pattern `http://[EXTERNAL-IP]:8080` to verify it works.

---

## Task 5. Delete the Cluster
To prevent incurring ongoing infrastructure charges for Compute Engine worker nodes once you finish testing, delete your resources.

1. Delete the GKE container cluster running via the CLI:
   ```bash
   gcloud container clusters delete lab-cluster --zone=us-east1-c
   ```
2. When prompted with `Do you want to continue (Y/n)?`, type **`Y`** and press **Enter** to confirm. 

---


