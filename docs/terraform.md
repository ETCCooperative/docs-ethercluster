# Terraform

[Terraform]() is infra-as-code to allow you to provision your cloud infrastructure in a way that's 
clear and easy to roll-back and version control.

This allows any changes you make to your cloud architecture to be reflected in code and saved, so
that any new changes you add can be tracked and debugged.

Here, we will go over installing Terraform and setting up a Google Kubernetes Engine cluster for it.

## Installation

First, we will need to download the Terraform binary for our appropriate OS from [here](https://www.terraform.io/downloads.html).

For this guide, I'll be going over the MacOS installation and setup, which is the same as on Linux.

After you download the zipped binary, you'll need to unzip it.
```sh
unzip terraform-xx.zip
```
Replace the terraform zip with the actual file name you downloaded. This will unzip the terraform executable.

You should see a terraform binary now. You can move it to an appropriate location and then add it to your path.

Here, I'll move it to my `$HOME` directory.
```sh
mv terraform /$HOME
```

Next, we will want to add it to our PATH so we can run it:
```sh
export PATH="$PATH:/$HOME/terraform"
```

Now, let's test out that it works in a new window.
```sh
terraform
```

You should see a list of commands that you can run, meaning it's working. If it's not working for you, follow
the Terraform [installation guide](https://learn.hashicorp.com/terraform/getting-started/install.html)


## Initialization

We will need to initialize Terraform in a directory to begin building our cluster.

We will make a directory called `ethercloud` as the place where we will initialize our Terraform in.

```sh
mkdir ethercloud && cd ethercloud
terraform init
```

You should see it respond in terminal with the following:
```sh
Terraform initialized in an empty directory!

The directory has no Terraform configuration files. You may begin working
with Terraform immediately by creating Terraform configuration files.
```

Great, now that Terraform is initialized, head over to the next section to begin building with it.
