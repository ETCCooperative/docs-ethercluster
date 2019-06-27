# Overview

This documentation guide is to help you build your own Ethercluster RPC infrastructure
step-by-step from scratch and learn a lot of DevOps tools used to build it.

We will be going over many different technologies and tools we will be using.

This design currently uses a Google-Cloud Cluster zone to building your infrastructure.
The interest in the future is to make Ethercluster a highly-available cluster across multiple 
cloud-providers and geographical locations.

## Motivation

Ethercluster began as a project for Ethereum Classic community due to frustrations at not having
a viable node infrastructure service that provides a reliable RPC endpoint.

All the major node providers such as Infura don't support ETC yet, so Ethercluster came about
from a desire to have a service like Infura that can scale as demand requires.

Since the ETC Cooperative is also a nonprofit, just providing this service to everyone would be costly
in the long run as the ETC network grows.

This is why we open-sourced the entire design! That way, you don't just have to rely on the endpoint we provide
but also build your own Ethercluster.

The designs in this guide target ETC and Kotti networks, but since we are using Parity as our client, literally 
any EVM-based network can be considered here. You just need to modify the volume size (ETH is nearly 2TB on full node). 
