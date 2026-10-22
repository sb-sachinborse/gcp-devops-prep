# GCP Compute Engine: VM Provisioning, Instance Groups, Autoscaling & Terraform

## 📌 Project Overview

This hands-on Google Cloud Platform (GCP) project demonstrates how to provision, configure, manage, and scale Virtual Machine (VM) instances using the `gcloud` CLI and Terraform.

The project progresses from creating a single VM to building scalable infrastructure with Managed Instance Groups (MIGs), startup scripts, and Infrastructure as Code (IaC).

## 🎯 Learning Objectives

- Configure GCP projects, regions, and zones.
- Create and manage Compute Engine VM instances.
- Provision multiple VMs using CLI commands.
- Create managed and unmanaged instance groups.
- Configure autoscaling and autohealing.
- Automate VM configuration with startup scripts.
- Provision infrastructure using Terraform.
- Verify resources and clean up to control costs.

## 🏗️ Architecture

```text
                  Google Cloud Platform
                           |
                     Compute Engine
                           |
             +-------------+-------------+
             |                           |
       Standalone VMs              Instance Groups
                                         |
                               +---------+---------+
                               |                   |
                          Unmanaged MIG*       Managed MIG
                          Instance Group       Instance Group
                               |                   |
                         Existing VMs       Instance Template
                                                   |
                                           Multiple VM Instances
                                                   |
                                            Autoscaling
                                                   |
                                         Health Checks / Repair

                  Infrastructure as Code
                           |
                        Terraform
                           |
                  VM / Network Resources
```

*An unmanaged instance group is not a MIG; it does not automatically maintain or replace instances.*

## 🛠️ Prerequisites

- Google Cloud account and an active project.
- Billing enabled where required.
- Google Cloud Shell with `gcloud` installed.
- Compute Engine API enabled.
- Appropriate IAM permissions.
- Terraform installed for the Terraform section.

Enable the Compute Engine API:

```bash
gcloud services enable compute.googleapis.com
```

Set your project:

```bash
gcloud config set project YOUR_PROJECT_ID
```

Verify:

```bash
gcloud config get-value project
```

## 1. Configure Region and Zone

Set the default region and zone:

```bash
gcloud config set compute/region asia-east1
gcloud config set compute/zone asia-east1-b
```

Create reusable environment variables:

```bash
export REGION=asia-east1
export ZONE=asia-east1-b
```

Verify:

```bash
echo $REGION
echo $ZONE
```

- **Region:** Geographical location for resources.
- **Zone:** Deployment location within a region.

## 2. Create a Single VM Instance

Create a VM named `gcelab2`:

```bash
gcloud compute instances create gcelab2 \
  --machine-type=e2-medium \
  --zone="$ZONE"
```

Verify:

```bash
gcloud compute instances list
```

Inspect details:

```bash
gcloud compute instances describe gcelab2 \
  --zone="$ZONE"
```

Connect using SSH:

```bash
gcloud compute ssh gcelab2 --zone="$ZONE"
```

Exit the remote shell:

```bash
exit
```

## 3. Create Multiple VM Instances

Create three VMs using a Bash loop:

```bash
for i in 1 2 3; do
  gcloud compute instances create "web-vm-$i" \
    --machine-type=e2-medium \
    --zone="$ZONE"
done
```

Verify the VMs:

```bash
gcloud compute instances list
```

Delete the VMs when finished:

```bash
gcloud compute instances delete \
  web-vm-1 web-vm-2 web-vm-3 \
  --zone="$ZONE"
```

**Use case:** Learning bulk provisioning and automating repetitive infrastructure tasks.

## 4. Create an Unmanaged Instance Group

An unmanaged instance group lets you group existing VMs for organizational purposes and supported load-balancing configurations. It does not automatically create, repair, or scale its member instances.

Create two VMs:

```bash
gcloud compute instances create \
  web-unmanaged-1 web-unmanaged-2 \
  --machine-type=e2-medium \
  --zone="$ZONE"
```

Create the unmanaged instance group:

```bash
gcloud compute instance-groups unmanaged create web-unmanaged-group \
  --zone="$ZONE"
```

Add the VMs:

```bash
gcloud compute instance-groups unmanaged add-instances \
  web-unmanaged-group \
  --zone="$ZONE" \
  --instances=web-unmanaged-1,web-unmanaged-2
```

List the group members:

```bash
gcloud compute instance-groups unmanaged list-instances \
  web-unmanaged-group \
  --zone="$ZONE"
```

**Key point:** You manage the VM lifecycle yourself. Adding a VM to an unmanaged group does not transfer lifecycle management to the group.

## 5. Create a Managed Instance Group (MIG)

A Managed Instance Group uses an instance template to create and manage identical VM instances. It supports features such as autoscaling, autohealing, and rolling updates.

### Step 5.1 — Create a startup script

Create a file named `startup.sh`:

```bash
cat > startup.sh <<'EOF'
#!/bin/bash
set -e

apt-get update
apt-get install -y apache2

cat > /var/www/html/index.html <<'HTML'
<!DOCTYPE html>
<html>
<head>
  <title>GCP Managed Instance Group</title>
</head>
<body>
  <h1>Hello from Google Cloud Compute Engine!</h1>
  <p>Web server deployed using a VM startup script.</p>
</body>
</html>
HTML

systemctl enable apache2
systemctl restart apache2
EOF
```

The script installs Apache and creates a simple web page when the VM starts.

### Step 5.2 — Create an instance template

```bash
gcloud compute instance-templates create web-template \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=startup.sh \
  --tags=http-server
```

The template defines the configuration used for new VM instances.

### Step 5.3 — Create a managed instance group

```bash
gcloud compute instance-groups managed create web-mig \
  --template=web-template \
  --size=2 \
  --zone="$ZONE"
```

Verify the instances:

```bash
gcloud compute instance-groups managed list-instances \
  web-mig \
  --zone="$ZONE"
```

Inspect the group:

```bash
gcloud compute instance-groups managed describe web-mig \
  --zone="$ZONE"
```

## 6. Configure Autoscaling

Autoscaling automatically adjusts the number of VM instances according to a configured metric and target utilization.

Configure the MIG to maintain between 2 and 5 instances based on average CPU utilization:

```bash
gcloud compute instance-groups managed set-autoscaling web-mig \
  --zone="$ZONE" \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-cpu-utilization=0.60 \
  --cool-down-period=60
```

Inspect the autoscaling configuration:

```bash
gcloud compute instance-groups managed describe web-mig \
  --zone="$ZONE"
```

**How it works:**

- Minimum replicas: 2
- Maximum replicas: 5
- Target average CPU utilization: 60%
- Scale-out or scale-in decisions depend on the autoscaler, workload, and configured limits.

Scaling is not necessarily instantaneous. The autoscaler needs time to observe demand, create VMs, and initialize applications.

> **Important:** Autoscaling controls the number of instances, not the distribution of incoming HTTP traffic. For a production web service, configure a suitable load balancer and health checks as well.

## 7. Configure Autohealing

Autohealing can recreate instances when their health checks indicate that they are unhealthy.

Create a health check:

```bash
gcloud compute health-checks create http web-health-check \
  --port=80
```

Attach the health check to the managed instance group:

```bash
gcloud compute instance-groups managed update web-mig \
  --zone="$ZONE" \
  --health-check=web-health-check \
  --initial-delay=300
```

The initial delay gives the application time to start before autohealing decisions begin.

For production, configure health-check firewall access and suitable health-check thresholds. Autohealing is different from load-balancer health checking, although both can use health-check resources.

## 8. Configure Firewall Access for the Web Server

To test the Apache web server, allow HTTP traffic on port 80 using a firewall rule:

```bash
gcloud compute firewall-rules create allow-http-web \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server
```

This example assumes the VMs use the default VPC network. Configure the correct VPC if your project uses a custom network.

**Security note:** This rule exposes HTTP to the internet. For production workloads, restrict access where appropriate and use HTTPS and a properly configured load balancer.

Retrieve a VM's external IP:

```bash
gcloud compute instances list
```

Open `http://EXTERNAL_IP` in your browser.

## 9. Create a VM Using Terraform

Terraform is an Infrastructure as Code (IaC) tool that lets you define cloud infrastructure in configuration files and manage it consistently.

### Step 9.1 — Install Terraform

Install Terraform by following the official instructions:

https://developer.hashicorp.com/terraform/install

Verify:

```bash
terraform version
```

Authenticate using Google Cloud Shell or Application Default Credentials:

```bash
gcloud auth application-default login
```

For Cloud Shell, follow the authentication instructions displayed by Google Cloud. In automated environments, prefer workload identity federation or an appropriately scoped service account instead of user credentials.

### Step 9.2 — Create a project directory

```bash
mkdir gcp-terraform-vm
cd gcp-terraform-vm
```

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}

variable "project_id" {
  description = "Google Cloud project ID"
  type        = string
}

variable "region" {
  description = "Google Cloud region"
  type        = string
  default     = "asia-east1"
}

variable "zone" {
  description = "Google Cloud zone"
  type        = string
  default     = "asia-east1-b"
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

resource "google_compute_instance" "vm" {
  name         = "terraform-vm"
  machine_type = "e2-medium"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 20
      type  = "pd-balanced"
    }
  }

  network_interface {
    network = "default"
    access_config {}
  }

  labels = {
    environment = "learning"
    managed_by  = "terraform"
  }
}

output "vm_name" {
  value = google_compute_instance.vm.name
}

output "vm_zone" {
  value = google_compute_instance.vm.zone
}

output "vm_external_ip" {
  value = google_compute_instance.vm.network_interface[0].access_config[0].nat_ip
}
```

### Step 9.3 — Configure Terraform variables

Create `terraform.tfvars`:

```hcl
project_id = "YOUR_PROJECT_ID"
region     = "asia-east1"
zone       = "asia-east1-b"
```

Replace the placeholder with your actual project ID.

### Step 9.4 — Initialize Terraform

```bash
terraform init
```

This downloads the required provider and initializes the working directory.

### Step 9.5 — Format and validate

```bash
terraform fmt
terraform validate
```

### Step 9.6 — Review the execution plan

```bash
terraform plan
```

Review the resources Terraform proposes to create before proceeding.

### Step 9.7 — Create the VM

```bash
terraform apply
```

Review the plan and type `yes` when prompted.

Retrieve the outputs:

```bash
terraform output
```

### Step 9.8 — Destroy the Terraform-managed resources

When finished with this standalone Terraform lab:

```bash
terraform destroy
```

Review the plan and confirm when prompted.

**Important:** Terraform only manages the resources declared in or tracked by its state. It will not automatically delete VMs created manually using `gcloud`.

## 10. Recommended Repository Structure

```text
gcp-compute-engine-lab/
├── README.md
├── gcloud/
│   ├── create-vm.sh
│   ├── create-multiple-vms.sh
│   ├── create-unmanaged-group.sh
│   ├── create-mig.sh
│   └── startup.sh
├── terraform/
│   └── gcp-terraform-vm/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars.example
│       └── .gitignore
└── screenshots/
    ├── vm-instance.png
    ├── managed-instance-group.png
    └── terraform-apply.png
```

For a cleaner implementation, move the Terraform variable and output declarations from `main.tf` into `variables.tf` and `outputs.tf`, respectively. Terraform loads all `.tf` files in the directory automatically.

Suggested `.gitignore` for Terraform:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfplan
crash.log
```

Keep real credentials, private keys, and sensitive variable files out of GitHub. Commit `.terraform.lock.hcl` to record provider selections.

## 11. Verification Checklist

- [ ] Configured the GCP project, region, and zone.
- [ ] Created and verified a standalone VM.
- [ ] Created multiple VM instances.
- [ ] Created an unmanaged instance group and added VMs.
- [ ] Created an instance template and managed instance group.
- [ ] Configured autoscaling.
- [ ] Configured autohealing and health checks.
- [ ] Deployed a web server using a startup script.
- [ ] Created a VM using Terraform.
- [ ] Reviewed Terraform plans before applying changes.
- [ ] Cleaned up lab resources.

## 🧠 Key Learnings

This project demonstrates practical skills in cloud infrastructure provisioning, VM lifecycle management, instance-group operations, autoscaling, startup automation, health monitoring, and Infrastructure as Code.

It provides a foundation for more advanced DevOps projects involving Terraform modules, CI/CD pipelines, load balancing, private networking, IAM least privilege, and production-grade GCP infrastructure.

## 📚 Official Documentation

- [Compute Engine VM Instances](https://cloud.google.com/compute/docs/instances)
- [Unmanaged Instance Groups](https://cloud.google.com/compute/docs/instance-groups/ungrouped-instances)
- [Managed Instance Groups](https://cloud.google.com/compute/docs/instance-groups)
- [MIG Autoscaling](https://cloud.google.com/compute/docs/autoscaler)
- [MIG Autohealing](https://cloud.google.com/compute/docs/instance-groups/autohealing-in-migs)
- [Startup Scripts](https://cloud.google.com/compute/docs/instances/startup-scripts)
- [Terraform on Google Cloud](https://cloud.google.com/docs/terraform)
- [Google Provider for Terraform](https://registry.terraform.io/providers/hashicorp/google/latest/docs)

---

**Project category:** Google Cloud | Compute Engine | Linux | Cloud Automation | Terraform | Infrastructure as Code

**Suggested commit message:**

`feat: add GCP VM automation, MIG autoscaling, and Terraform examples`
