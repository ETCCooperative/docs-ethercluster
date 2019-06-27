# Google Cloud Shell

Now, I want to show you how you can SSH into a Google Cloud Shell which you can use to connect to your cluster
and start building out your Kubernetes architecture.

You can also use the Google Cloud Shell console inside Google Cloud if you don't want to SSH.

## GCloud Installation

First, we will need to install the `gcloud`. I'll be running the installation on MacOS, but
feel free to use your preferred OS from the guide [here](https://cloud.google.com/sdk/install)

We install GCloud SDK with the following command:
```sh
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
```

We test it works with:
```
gcloud init
```

Great, it's now installed.


## SSH into Google Cloud Shell

To SSH into our GCloud Shell, run the following command:
```sh
gcloud alpha cloud-shell ssh
```

The following output is shown:
```sh
Starting your Cloud Shell machine...
Waiting for your Cloud Shell machine to start...done.
Warning: Permanently added '[devshell-vm-d52c6025-6e5b-4014-9710-0bc0b3395ab6.cloudshell.dev]:6000,[35.231.187.159]:6000' (RSA) to the list of known hosts.
Welcome to Cloud Shell! Type "help" to get started.
Your Cloud Platform project in this session is set to ethercluster.
Use “gcloud config set project [PROJECT_ID]” to change to a different project.
yaz@cloudshell:~ (ethercloud)$
```

This shows that we successfully SSH-ed into our ethercloud project that we have set up before. 

Great, now we are finally ready for Kubernetes deployment!
