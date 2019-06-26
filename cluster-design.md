# Cluster Design

In order to build our Ethercluster, we will need to think about which client we want to use and how much volume to assign it.

We will also need to think about how to organize the project in the cluster.

## Namespace

We first need to define a namespace for our ethercluster. Namespaces in Kubernetes allow us to assign the a name for specific projects
we are working on inside Kubernetes. It's useful if you want to organize your cluster between `dev` and `prod` namespaces for example.

Here, we will just be using it for `ethercluster` to make it easier to see everything.

To create a namespace, I'll be writing up a YAMl config file for Kubernetes.

Each Kubernetes manifest file has three things:
* apiVersion (to specify which API to use)
* kind (to determine what Kubernetes component the file is)
* metadata (extra information about the manifest)

We also have a `spec` section to specify how we want the manifest behaves, which we will see when we build our cluster.

Picking up from where we left off in Google Cloud Shell page, let's see if `kubectl` is running. `kubectl` is a command line
application that uses the Kubernetes API to interact with your cluster.

We first connect to our cluster from inside Google Cloud Shell via: 
```sh
gcloud container clusters get-credentials ethercloud --zone us-central1-c --project ethercloud
```

Next, we test `kubectl` is working by running:
```
kubectl
```

The output should look something like this:
```sh
kubectl controls the Kubernetes cluster manager.

Find more information at: https://kubernetes.io/docs/reference/kubectl/overview/

*
*
*

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
```

The output displayed isn't the complete one, but generally if it shows something like what's above, then Kubectl is working.

Now, let's build the namespace.

We will first define the Namespace manifest file:
```sh
vim ethercluster-namespace.yml
```

Inside the YAML file, we add the following code:
```yml
apiVersion: v1
kind: Namespace
metadata:
    name: ethercluster 
    labels:
        name: ethercluster 
```

This specifies that our manifest is `kind` of `Namespace`, with a name of `ethercluster`.

Let's now create the namespace now that we have written the manifest for it.

```sh
kubectl apply -f ethercluster-namespace.yml
```

This should output the following:
```sh
namespace/ethercluster created
```

Great, now that we have the namespace defined, we can proceed with creating the rest of our cluster components.

## Volumes

We will need to specify a [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) for our Deployment volume. Here, we will use SSD since it's best for syncing 
clients.

```
vim classic-storage-class.yml
```

Inside the manifest, we will specify the name of our StorageClass
```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: classic-ssd
  namespace: ethercluster
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zones: us-central1-c
reclaimPolicy: Retain
```

Notice how we specified the provisioner to be `gce-pd`. It's to specify the Cloud Provider type of disk we want.
We then specify it to be a persistent disk SSD `pd-ssd` and the zone is `us-central1-c` which is the same one we specified
in Terraform, which is where our GKE cluster was created.

Let's create the StorageClass via `kubectl apply -f classic-storage-class.yml`

This will create the StorageClass in Google Cloud which we will use after when doing a Deployment.


## Service

In Service, we will need to specify the ports we are interested in obtaining from our node. For the purpose of a
public RPC endpoint, we will need port 8545 which is the default RPC port. If you need something more custom, like WebSockets,
then port 8546 is the one you want. Here, we will only go over 8545. We add 8080 for default and 443 for SSL.

```sh
vim classic-service.yml
```

Inside the manifest, we add the following code:

```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: classic
  name: classic
  namespace: ethercluster
  annotations:
    cloud.google.com/app-protocols: '{"my-https-port":"HTTPS","my-http-port":"HTTP"}'
spec:
  selector:
    app: classic
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 80
  - name: rpc-endpoint
    port: 8545
    protocol: TCP
    targetPort: 8545
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  type: LoadBalancer
  sessionAffinity: ClientIP
```

If you notice, we specify the `kind` to be Service here. We call the service `classic` which will allow it to auto-discover 
the deployment after with the same name.

Notice how we have `port` and `targetPort` specified. It tells the Service we want to this Service's port 8545 to route to the 
container's port 8545.

We specify the type to be `LoadBalancer` to expose it and allow for selecting between the different nodes we will create with Parity.
We also add a `sessionAffinity` so that if you connect to a node assigned by the load balancer, the next time you connect to it,
load balancer will reconnect you to the same node.

Now, let's instantiate the service:
```sh
kubectl apply -f classic-service.yml
```

This will create the service.

Now, run the following to get all components of your cluster:
```sh
kubectl get all -n ethercluster
```

This will output the following:
```sh
NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                                       AGE
service/classic             LoadBalancer   10.00.00.00   <pending>      8080:30003/TCP,8545:30002/TCP,443:30001/TCP   1m
```

**NOTE**: Actual values displayed above have been modified for example purposes only for this guide.

You notice a `<pending>` on the External-IP. This is because Kubernetes is still creating an endpoint to expose, which takes a
few minutes. While this happens, let's get on with our deployment.


## Deployment 

For the rest of the exercise, we will go over the code found in this [repository](https://github.com/ethereum-classic-cooperative/ethercluster).

You can clone this repository inside your Google Cloud Shell with the following command:
```sh
git clone https://github.com/ethereum-classic-cooperative/ethercluster
cd ethercluster
```
Now we are inside our ethercluster directory.

Before running our deployment, we need to figure out what arguments we need for our Docker container image. 

We will be using [Parity](https://github.com/paritytech/parity-ethereum) for our client image here.

If you're familiar with Docker, you know we can run a Parity container the following way:
```sh
docker run -ti -p 8545:8545 parity/parity:latest --chain classic
```

This runs the latest Parity image via docker and exposes the port 8545 for RPC and runs the Ethereum Classic chain.

Things like the `--chain classic` can be added to the deployment as well.

If you go to `deployments/classic-statefulset.yml`, you'll see that the file has the `kind` value of [`StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/).
This is important since it specifies we want a stateful application that guarantees ordering and uniqueness across any rescheduling.
Imagine we didn't use a statefulset here. If we deploy our container to Kubernetes, Parity begins syncing the chain from the beginning.
Kubernetes can then choose to restart Pods at intervals to ensure updates. What happens in this situation is that the restart will also
cause Parity to resync from the beginning, which isn't what we desire. It's why StatefulSet is the desired Deployment here.

```yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: classic
  namespace: ethercluster
  labels:
    app: classic
```

This is the beginning of our StatefulSet deployment file. It specifies the correct namespace for Ethercluster, names our Deployment 
`classic` in order to be able to identify what network it is, and points the `kind` to StatefulSet.

This isn't the only contents of the file, we will go on and define the `specs`.

In the specs, you'll notice we have the `replicas` to be 3. This means we will be running 3 Parity nodes, which we will LoadBalance.
In `containers` section, we specify the image `parity` from Parity's Dockerhub endpoint.

We pass in the values to that containers such as `chain=classic`. This specifies that we want the Ethereum Classic chain to be our
default network to run. This is how Kubernetes can specify the arguments for the container like Docker does in the previous example.

We also specify we want the ports 8545 since we want to expose the RPC. We have some readinessProbe and livenessProbe in order to do 
health checks on the Probe. It happens by doing an HTTP GET request on port 8545 of the container on the endpoint `/api/health`, which
checks if the Parity node is fully synced or not. If it's not synced up yet, it returns back a 503, otherwise it will return a 200, thus
passing the health check.

We also specify a `volumeClaimTemplates` for this Deployment of 50 GB, which is what will be needed to run a full ETC node. 
If you want to instead run an ETH node, you'll need to adjust the value appropriately (300 GB to stay on the safe side).

Now, we will instantiate the deployment:
```sh
kubectl apply -f deployments/classic-statefulset.yml
```

This will instantiate your StatefulSet. Now let's see it in action.

```sh
kubectl get all -n ethercluster
```

This will output the following:

```sh
NAME                                     READY   STATUS    RESTARTS   AGE
pod/classic-0                            2/2     Running   0          1m
pod/classic-1                            2/2     Running   0          1m
pod/classic-2                            2/2     Running   0          1m

NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                                       AGE
service/classic             LoadBalancer   10.00.00.00     109.01.01.01      8080:30003/TCP,8545:30002/TCP,443:30001/TCP   1m

NAME                       READY   AGE
statefulset.apps/classic   3/3     1m
```

Note that the age shown above might not be exact to what you get since it's still creating each Pod 1 by 1. Why do we have
three pods? It's because we specifed our replica to be 3 in our deployment file.

Each pod is created by the Statefulset, where it's mounted to a volume from `classic-ssd` that we instantiated before, and then
each image of the containers are pulled and instantiated, and begin running, before the next pod is created.

We can inspect each pod specifically with the following command:

```sh
kubectl describe pod classic-0 -n ethercluster
```

This will output:

```sh
Name:               classic-0
Namespace:          ethercluster
Priority:           0
PriorityClassName:  <none>
Node:               gke-ether-cluster-my-node-pool-XXXXXXX
Start Time:         Wed, 26 Jun 2019 14:58:55 -0400
Labels:             app=classic
                    controller-revision-hash=classic-5866fc986
                    statefulset.kubernetes.io/pod-name=classic-0
Annotations:        <none>
Status:             Running
IP:                 10.2.2.10
Controlled By:      StatefulSet/classic
Containers:
  parity:
    Container ID:  docker://6e0adffcaebb98e285e3ee432bca2ddd64cd555a6207cb44057ce32676f05a5b
    Image:         parity/parity:v2.5.2-beta
    Image ID:      docker-pullable://parity/parity@sha256:3e347079e21de53333ddba0b351ed1c91954c95acc2c2a7ef815ed58f804baac
    Ports:         8545/TCP, 443/TCP
    Host Ports:    0/TCP, 0/TCP
    Args:
      --chain=classic
      --base-path=/classic-data
      --db-path=/classic-data/chains
      --keys-path=/classic-data/keys
      --jsonrpc-port=8545
      --jsonrpc-interface=0.0.0.0
      --jsonrpc-apis=web3,eth,net,rpc,parity
      --jsonrpc-hosts=all
      --no-ancient-blocks
      --no-serve-light
      --max-peers=250
      --snapshot-peers=50
    State:          Running
      Started:      Wed, 26 Jun 2019 14:59:05 -0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /classic-data from classic-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-m79bm (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  classic-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  classic-data-classic-0
    ReadOnly:   false
  default-token-m79bm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-m79bm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
``` 

**NOTE** Some of this data has been modified for the purpose of this guide. Now let's check if our Parity node is syncing yet.

We can use the `logs` command for that.

```sh
kubectl logs classic-0 parity -n ethercluster
```

This will output the following:
```sh
2019-06-26 18:59:05 UTC Starting Parity-Ethereum/v2.5.2-beta-ecbafb2-20190611/x86_64-linux-gnu/rustc1.35.0
2019-06-26 18:59:05 UTC Keys path /classic-data/keys/classic
2019-06-26 18:59:05 UTC DB path /classic-data/chains/classic/db/906a34e69aec8c0d
2019-06-26 18:59:05 UTC State DB configuration: fast
2019-06-26 18:59:05 UTC Operating mode: active
2019-06-26 18:59:05 UTC Configured for Ethereum Classic using Ethash engine
2019-06-26 18:59:05 UTC Listening for new connections on 127.0.0.1:8546.
2019-06-26 18:59:11 UTC Public node URL: enode://f61d86747ef4e1bc5d1ab879cf390217259a487f0f9784db099c293c84cef4aeff9fa40802323111fb6d5699e061b283fb78ed3647792ef140b79c44bd85448f@10.28.21.10:30303
2019-06-26 18:59:15 UTC Syncing snapshot 0/670        #0    4/25 peers      8 KiB chain  513 KiB db  0 bytes queue   15 KiB sync  RPC:  0 conn,    2 req/s,   24 µs
2019-06-26 18:59:21 UTC Syncing snapshot 1/670        #0    5/25 peers      8 KiB chain  513 KiB db  0 bytes queue   15 KiB sync  RPC:  0 conn,    2 req/s,   33 µs
2019-06-26 18:59:25 UTC Syncing snapshot 6/670        #0    6/25 peers      8 KiB chain  513 KiB db  0 bytes queue   15 KiB sync  RPC:  0 conn,    2 req/s,   33 µs
2019-06-26 18:59:31 UTC Syncing snapshot 9/670        #0    7/25 peers      8 KiB chain  513 KiB db  0 bytes queue   15 KiB sync  RPC:  0 conn,    2 req/s,   34 µs
```

Woohoo! Parity is now syncing our Ethereum Classic node.

We can test if it responds to RPC commands with the following (replace `IP` with your `External IP `
from `kubectl get services classic -n ethercluster`)

```
curl -v --data '{"method":"parity_nodeStatus","params":[],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST IP:8545
```

This should return that your node is still syncing as shown below:
```
{"jsonrpc":"2.0","error":{"code":-32001,"message":"Still syncing."},"id":1}
```

If your node is fully synced by then, it'll return
```
{"jsonrpc":"2.0","result":null,"id":1}
```

This means your clear to use your new RPC endpoint!

In the next section, we will go over securing your endpoint with SSL when using it publicly. It's not really needed if you want 
to use RPC only internally within your own infrastructure.
