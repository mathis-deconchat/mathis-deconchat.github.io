---
title: "GCP Cluster"
date: 2023-08-23T06:20:42+02:00
draft: false
icon: "code_blocks"
weight: 100
description: "Create and manage a GCP cluster with Terraform"

---
## Goals
Create a kubernetes cluster on Google Cloud Platform (GCP) with Terraform. Modify this cluster, deploy applications on it and destroy it.

## Prerequisites
Click links to install the following tools:
1. [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
2. [GCP](https://cloud.google.com/sdk/docs/install#deb) (Google) account with free credits not used
3. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

## GCP steps
For every steps, a simple google research will help you if you are lost or if you want to know more about a specific topic.

### Free credits
First, access the console 
[console](https://console.cloud.google.com/)
If you don't have the free credits activated, it will prompt you to do it. FOlow the different steps.

{{< alert context="warning" text="A credit card will be required, it's only to check if you are human (you can trust google I guess), it will **NEVER** charge you, unless you choose to do so after your credit are exprired. " />}}

### Create a project 
On the top-left corner, a dropdown menu will allow you to see current project (Only First Project for now, if your account is new), and to create a project. Create one called `tf-gke` or whatever you want.

Now, a pop-up will appear, select the project. In the same dropdown you used to create the project you should now see `tf-gke`
Be aware that the project id will contains a random string at the end. For example `tf-gke-***`<br><br>
[📕 Documentation](https://cloud.google.com/resource-manager/docs/creating-managing-projects)

### Connect gcloud SDK 
```bash
gcloud init 
```
It will prompt you to login with your google account. Then, it will ask you to select a project. Select the one you just created. In this case `tf-gke-***`

### Enable GCP Kubernetes API
{{< tabs tabTotal="2">}}
{{% tab tabName="Via CLI" %}}

```bash
gcloud services enable container.googleapis.com
```
{{% /tab %}}
{{% tab tabName="Via GCP website" %}}
Search "Kubernetes Engine API" in the top search bar. 
Then, click on "Enable", it might take some time
{{% /tab %}}
{{< /tabs >}}

### Zones and regions 
Depending on your localisation, you will have different zones and regions available. For example, I'm in France, so I have `europe-west1` and `europe-west3` available for regions and `europe-west1-b`, `europe-west1-c` and `europe-west1-d` for zones. 
To list all zones (and their regions) available for you, run the following command:
```bash
gcloud compute zones list
```
Update your project with the appropriate zone and region
```bash
gcloud compute project-info add-metadata \
   --metadata google-compute-default-region={REGION},google-compute-default-zone={ZONE}
```

{{< alert context="info" text="You can check your localisation with `gcloud config get-value compute/region` and `gcloud config get-value compute/zone`" />}}

### Node machine type 
First you need to choose a machine type for your nodes. 
You can list machine types available for you with the following command:
```bash
gcloud compute machine-types list --zone={ZONE}
```
For this tutorial, we will use `e2-medium` which is a 2 vCPU and 4GB RAM machine.
{{< alert context="info" text="You have a quota of machine when using free credit, don't choose high end machines. You'll get an explicit error if you do so" />}}

### Création du fichier de credentials
The terraform provider needs credentials to access your GCP account. For this example we will use the built-in method in gcloud SDK to generate a credentials file.
```bash
gcloud auth application-default login
```
It will open a browser window, select your account and click on "Allow". Then, you can close the browser window.
## Minimalist cluster
### Create cluster
Now, create a folder and add a `main.tf` file inside it and add this 15 lines of code:

```hcl {linenos=table,anchorlinenos=true}
provider "google" {
  project     = "tf-gke-396804"
  region      = "europe-west9"
}

resource "google_container_cluster" "gke_cluster" {
  name               = "gke-cluster"
  location           = "europe-west9"
  initial_node_count = 3

  node_config {
    machine_type = "e2-medium"
    disk_size_gb = 20
  }
}
```

{{< alert context="info" 
text="For simplicity purpose, set an alias `alias tf='terraform'` in your bash profile, or replace `tf` with `terraform` in the following commands." />}}

Then run the following commands:
```bash
tf init
tf plan
tf apply -auto-approve
```
Refer to the 
[intro]({{< ref "/docs/terraform/intro.md" >}}) 
if you don't understand what's happening and what result should you see.

After `tf apply -auto-approve`, it should take around 5-10 minutes before you see : 
```bash
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

We can check if the cluster is up and running with the following command:
```bash
gcloud container clusters list
```
And the output should be something like this:
```bash
NAME         LOCATION      MASTER_VERSION  MASTER_IP      MACHINE_TYPE  NODE_VERSION    NUM_NODES  STATUS
gke-cluster  europe-west9  1.27.3-gke.100  34.155.108.14  e2-medium     1.27.3-gke.100  3          RUNNING
```

### Get credentials
Now, we need to get the credentials of the cluster to be able to interact with it. 
```bash
gcloud container clusters get-credentials gke-cluster --region europe-west9
```
The output will be a file in ~/.kube/config. You can check it with `cat ~/.kube/config`

### Destroy cluster
Then destroy it with 
```bash
tf destroy -auto-approve
```
## Real world cluster
{{< alert context="info" text="You'll find many ressources online on this way to create a cluster. The following code is heavily inspired by [Hashicorp GKE](https://developer.hashicorp.com/terraform/tutorials/kubernetes/gke)"/>}}
### Intro
In this cluster, we will create the VPC (Virtual Private Cloud) and the subnetworks where the nodes will be deployed. If not specified, like the minimalist cluster, the nodes will be deployed in the default VPC and subnetworks. The reason why we create a new VPC is that we don't want to use the default one since it's shared with other users. We want to have our own VPC to be able to configure it as we want.
We will also create separate node pools, to be able to update nodes without restarting the whole cluster.

Create a new folder or erase the content of the `main.tf` from before.
Create the following file structure:
```bash
touch vpc.tf node-pool.tf variables.tf var.tfvars provider.tf backend.tf gke.tf
```
### State management
In the minimalist cluster example, the state was stored locally. It's not a good practice since it's not shared between the team. We will use a remote backend to store the state in a bucket on GCP.

First activate the API of Google Cloud Storage (GCS)
```bash
gcloud services enable storage.googleapis.com
```
Create a bucket to store the state of our infra. 

{{< alert context="info" text="The bucket name must be unique (on all GCP), so change the name in the following command" />}}
I'll add part of my project ID to the name to be sure it's unique.
```bash
gcloud storage buckets create gs://tf-state-gke-396804 --project tf-gke-396804 --location europe-west9
```
The output should be something like this:
```bash
Creating gs://tf-state-gke-396804/...
``` 
We also need object versionning on our bucket, so we can be able to restore a previous state if needed.
```bash
gcloud storage buckets update gs://tf-state-gke-396804 --versioning 
```
Output : 
```bash
Updating gs://tf-state-gke-396804/...                                       
  Completed 1   
```

Once done, add the following code to `backend.tf`:
```hcl
terraform {
 backend "gcs" {
   bucket  = {BUCKET_NAME}
   prefix  = "terraform/state/realworld"
 }
}
```

