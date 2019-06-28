# Setting up Google Kubernetes Engine

In this part, we will be using Terraform to instantiate a Google Kubernetes Engine (GKE)
which will be used to build our cluster.

GKE is a Google Cloud product that makes it much easier to manage your Kubernetes cluster
and get a high-level overview of it view a dashboard and friendly UI.

Don't worry too much about Kubernetes and what it does, it'll be covered in further sections.

## Provider

Here, we will need to specify the provider for our Terraform. I'll be using vim to write the code 
in Terminal, but feel free to use any IDE you like.

First, we need to tell Terraform which Cloud Provider we want and in what region.

Create a file called ethercloud.tf (.tf is the Terraform code file format)

```sh
vim ethercloud.tf
```

Now, inside the file, let's specify Google Cloud and the region. Add this code to your file:
```tf
provider "google" {
    project = "ethercloud"
    region = "us-central1"
    zone = "us-central1-c"
}
```

What we did here was specify `provider` as `google`, gave it a project name `ethercloud`, 
and then added the region `us-central-1` and the zone `us-central1-c`. You don't have to use
the same names and regions, but this will be what I use for the purpose of this guide.

Save the file.

## GKE

**Note** If you want the entire code in this section, checkout the Github repository [here](https://github.com/ethereum-classic-cooperative/ethercloud/blob/master/gcloud/ethercloud.tf)

Now, let's add the following code chunk to our `ethercloud.tf` file
```tf
resource "google_container_cluster" "primary" {
    name = "ethercluster"

    remove_default_node_pool = true
    initial_node_count = 1

    master_auth {
        username = ""
        password = ""
    }
}
```

This specifies that we want a GKE cluster we will call `primary`. We also specify the cluster name as `ethercluster`
We add an initial node count of 1, but we can't specify our own Kubernetes node pool first, so we use the initial node count
and then we delete it. We have our `master_auth` set to empty because we don't want to use any custom auth.

Next, add the next line of code to the file.

```
resource "google_container_node_pool" "primary_preemptible_nodes" {
    name = "my-node-pool"
    cluster = "${google_container_cluster.primary.name}"
    node_count = 3

    node_config {
        preemptible = true
        machine_type = "n1-standard-1"

        metadata = {
            disable-legacy-endpoints = "true"
        }

        oauth_scopes = [
            "https://www.googleapis.com/auth/logging.write",
            "https://www.googleapis.com/auth/monitoring",
        ]
    }
}
```

Here we are basically specifying the node pool with 3 nodes. Three nodes means three Google Compute Engine instances
where we will be hosting out Kubernetes cluster. We also specify the type of instance to be `n1-standard-1`.

For more information on this setup, checkout the guide from Terraform [here](https://www.terraform.io/docs/providers/google/r/container_cluster.html).

Moving forward, we want to add the Cluster certificate and a static IP address in case you chose to expose your RPC endpoint publicly
over SSL. It'll be easier to assign it to a static IP address you created on GKE so it doesn't change addresses every time you modify it.

Add the rest of the code chunk to your `ethercloud.tf` file:
```tf
output "client_certificate" {
    value = "${google_container_cluster.primary.master_auth.0.client_certificate}"
}

output "client_key" {
    value = "${google_container_cluster.primary.master_auth.0.client_key}"
}

output "cluster_ca_certificate" {
    value = "${google_container_cluster.primary.master_auth.0.cluster_ca_certificate}"
}

resource "google_compute_address" "ip_address" {
    name = "ethercluster-address"
}
```

Here, we basically specify the client key and certificate, as well as cluster certificate and ip_address. 
We will call this address `ethercluster-address`.

Great, we finally have the entire code we need to get started. Let's start terraforming!


## Terraform Plan and Apply

Now, let's see how this file gets generated by Terraform.

We first will need to run the following command in the same directory as our Terraform file.
```tf
terraform plan
```

Terraform plan allows you to see what changes Terraform will do to your file without making the changes.
It also is helpful for catching bugs.

Terraform begins analyzing your code and comes out with an output as shown here:
```sh
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_address.ip_address will be created
  + resource "google_compute_address" "ip_address" {
      + address_type       = "EXTERNAL"
      + name               = "ethercluster-address"
      *
    }

  # google_container_cluster.primary will be created
  + resource "google_container_cluster" "primary" {
      + name                        = "ether-cluster"
      + network                     = "default"
      *
    }

  # google_container_node_pool.primary_preemptible_nodes will be created
  + resource "google_container_node_pool" "primary_preemptible_nodes" {
      + cluster             = "ether-cluster"
      + id                  = (known after apply)
      *
      + management {
          * 
        }

      + node_config {
            *
        }
    }

Plan: 3 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------
```

Note, this isn't the exact output, it'll have a lot more data in it with keys, but the values will say `(known after apply)`.
This tells us that Terraform won't have the exact values for the configuration of our cloud for most of the keys until
after we actually create our architecture on Google Cloud.

Let's do that then.

You can apply the plan by running the following:
```sh
terraform apply
```

Now, since you already have your project json file from Google Cloud for authing added to your PATH in previous sections,
Terraform can use it to deploy your cloud architecture for you. It'll go through the same output you saw in plan but with
an execution plan and then start creating your instances. It'll take a little time to creates, so go have a coffee break
while you wait.

The output will look something like the following.

```sh
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:
*
*
*
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

Great, Terraform has created your infrastructure architecture. It's time to SSH into your Google Cloud Shell to see what you can do.