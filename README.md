# Terraform Template for Folding@home on GCP

Folding@home is simulating the dynamics of COVID-19 proteins to hunt for new therapeutic opportunities.
This template is provided to easily run Folding@home on Google Cloud, and help increase number of simulations done.
You can use this Terraform script to automatically deploy one or more Folding@home clients on GCP. The template creates the instance template with the Folding@home binaries, a managed instance group to uniformly deploy as many clients as specified by user, network firewall rules, and a Cloud NAT gateway for internet access without requiring public IPs, all in an existing or newly created network as specified by user.

This is not an officially supported Google product. Terraform templates for Folding@home are developer and community-supported. Please don't hesitate to open an issue or pull request.

[![button](http://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/GoogleCloudPlatform/terraform-folding-at-home&page=shell&tutorial=README.md)

### Prerequisites
* GCP project to deploy to.
* Optional: Existing network to deploy resources into.
* Sufficient resource quota in your GCP project and in the region you intend to deploy to. We will deploy a fixed-size managed instance group (MIG) with preemptible VMs which can be terminated (and replaced) at any time, but run at much lower price than normal instances. We will attach GPUs on each VM for workload acceleration so we also need to make sure we have GPUs quota.
  * Visit https://cloud.google.com/compute/quotas
  * Under Location, search and select a location such as "us-east1"
  * Under Metrics, search for "CPU" and select "CPUs" and "Preemptible CPUs". If you do not have Preemptible CPUs quota, Compute Engine will still use regular CPUs quota to launch preemptible VM instances.
  * Under Metrics, search for "GPU" and select all GPUs except "Commmitted.." ones and the "...Virtual Workstation GPUs". If you do not have Preemptible GPUs quota, Compute Engine will still use regular GPU quotas to add GPUs to preemptible VM instances.
  * This helps you determine (1) which GPU devices are most available and (1) how many spare CPU cores there are. Using this starting quota, you can now determine the upper bound of your MIG size (i.e. the number of VMs each running a Folding@home client) you can deploy, and whether you can attach GPUs (and which GPU device). For example, below is a screenshot of a newly created project with a starting quota in 'us-east1' region of 72 CPU cores and 4 Preemptible Nvidia T4 GPUs. In that case, one might opt with a MIG of size 4, where each worker node is a Preemptible n1-highcpu-8 with a T4 GPU attached, so a total of 4*8=32 CPUs and 4 GPUs. If desired, you can request for more quota (including separate and additional quota for preemptible CPUs/GPUs), by selecting the specific quota(s), clicking on 'Edit Quotas', and entering the requested 'New quota limit'.

![Compute Engine Quota Screenshot](./img/cpu_gpu_quota.png)

### Configurable Parameters

Parameter | Description | Default
--- | --- | ---
project | Id of the GCP project to deploy to |
region | Region for cloud resources | 
zones | One or more zones for cloud resources. If not set, up to three zones in the region are used to distributed instances depending on number of instances.<br>**Note on GPU:** Not all zones support GPUs. If running with GPUs, you should specify explicit list of zones (available for you in your region) that support your selected GPU model. [See list of zones per GPU model](https://cloud.google.com/compute/docs/gpus/#gpus-list)
create_network | Boolean to create a new network | true
network | Network to deploy resources into. It is either: <br>1. Arbitrary network name if create_network is set to true  <br>2. Existing network name if create_network is set to false | fah-network
subnetwork | Subnetwork to deploy resources into It is either: <br>1. Arbitrary subnetwork name if create_network is set to true  <br>2. Existing subnetwork name if create_network is set to false | fah-subnetwork
subnetwork_cidr | CIDR range of subnetwork | 192.168.0.0/16
fah_worker_image | Docker image to use for Folding@home client | stefancrain/folding-at-home:latest
fah_worker_count | Number of Folding@home clients or GCE instances | 3
fah_worker_type | Machine type to run Folding@home client on.<br>**Note on GPU:** only general-purpose N1 machine types currently support GPUs | n1-highcpu-8
fah_team_id | Team id for Folding@home client. Defaults to [F@h team Google or 446](https://stats.foldingathome.org/team/446) | 446
fah_user_name | User name for Folding@home client | Anonymous
<br>

### Getting Started

#### Requirements
* Terraform 0.12+

#### Setup working directory

1. Copy placeholder vars file `variables.yaml` into new `terraform.tfvars` to hold your own settings.
2. Update placeholder values in `terraform.tfvars` to correspond to your GCP environment and desired Folding@home settings. See [list of input parameters](#configurable-parameters) above.
3. Initialize Terraform working directory and download plugins by running `terraform init`.
4. Provide credentials to Terraform to be able to provision and manage resources in your project. See [adding credentials docs](https://www.terraform.io/docs/providers/google/guides/getting_started.html#adding-credentials) to supply a GCP service account key.

#### Deploy Folding@home instances

```shell
$ terraform plan
$ terraform apply
```

#### Access Folding@home process

Once Terraform completes:

1. Confirm Folding@home instance group has been created with correct number of instances
  * Navigate to Compute Enginer -> Instance groups: `https://console.cloud.google.com/compute/instanceGroups/list`
  * Click on the newly created instance group to view its details
  * Confirm number of instances created. Take note of one the instances names and corresponding zone

2. Access one of the new instances via CLI.
  * First, make sure you have IAP SSH permissions for your instances by [following these instructions](https://cloud.google.com/nat/docs/gce-example#step_4_create_ssh_permissions_for_your_test_instance)
  * Type `gcloud compute ssh [INSTANCE_NAME] --zone [INSTANCE_ZONE]` to SSH to the instance you took note previously. Since instances are created without external IP, this will default to using IAP access.

3. View Folding@home container logs
  * Once logged in, retrieve container name via `docker ps`
  * Type `docker logs -tf [CONTAINER_NAME]` to tail the logs and confirm its operation
 
### TODOs

* Fix GPU passthrough. Example error: `No compute devices matched GPU #0 NVIDIA:7 TU104GL [Tesla T4].  You may need to update your graphics drivers.`
* Fix logging to Stackdriver for quick monitoring & troubleshooting
* Scale down to 1 when no jobs available
* Scale down to 0 when no jobs available for extended time. Spin back up periodically.

### Support

This is not an officially supported Google product. Terraform templates for Folding@home are developer and community-supported. Please don't hesitate to open an issue or pull request.

### Copyright & License

Copyright 2020 Google LLC

Terraform templates for Folding@home are licensed under the Apache license, v2.0. Details can be found in [LICENSE](./LICENSE) file.
