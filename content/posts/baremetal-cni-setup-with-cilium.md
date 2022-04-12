---
title: "Baremetal CNI Setup with Cilium"
date: 2022-01-21T19:20:27+02:00
tags: ["kubernetes", "cni", "baremetal"]
draft: false
---

In a freshly set up Kubernetes cluster, we need a so-called CNI. This CNI is not always present after installation.

## What is a Container Network Interface (CNI)?
CNI is a network framework that allows the dynamic configuration of networking resources through a group of Go-written specifications and libraries. The specification mentioned for the plugin outlines an interface that would configure the network, provisioning the IP addresses, and mantain multi-host connectivity.

In the Kubernetes context, the CNI seamlessly integrates with the kubelet to allow automatic network configuration between pods using an underlay or overlay network. An underlay network is defined at the physical level of the networking layer composed of routers and switches. In contrast, the overlay network uses a virtual interface like VxLAN to encapsulate the network traffic.

Once the network configuration type is specified, the runtime defines a network for containers to join and calls the CNI plugin to add the interface into the container namespace and allocate the linked subnetwork and routes by making calls to IPAM (IP Address Management) plugin.

In addition to Kubernetes networking, CNI also supports Kubernetes-based platforms like OpenShift to provide a unified container communication across the cluster through software-defined networking (SDN) approach.

### What is Cilium?
Cilium is an open-source, highly scalable Kubernetes CNI solution developed by Linux kernel developers. Cilium secures network connectivity between Kubernetes services by adding high-level application rules utilizing eBPF filtering technology. Cilium is deployed as a daemon `cilium-agent` on each node of the Kubernetes cluster to manage operations and translates the network definitions to eBPF programs.

The communication between pods happens over an overlay network or utilizing a routing protocol. Both IPv4 and IPv6 addresses are supported for cases. Overlay network implementation utilizes VXLAN tunneling for packet encapsulation while native routing happens through unencapsulated BGP protocol.

Cilium can be used with multiple Kubernetes clusters and can provide multi CNI features, a high level of inspection,pod-to-pod connectivity across all clusters.

Its network and application layer awareness manages packet inspection, and the application protocol packets are using.

Cilium also has support for Kubernetes Network Policies through HTTP request filters. The policy configuration can be written into a YAML or JSON file and offers both ingress and egress enforcements. Admins can accept or reject requests based on the request method or path header while integrating policies with service mesh like Istio.

### Preparation
For the installation we need the CLI from Cilium.
We can install this with the following commands:

**Mac OSx**
```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-darwin-amd64.tar.gz{,.sha256sum}
shasum -a 256 -c cilium-darwin-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-darwin-amd64.tar.gz /usr/local/bin
rm cilium-darwin-amd64.tar.gz{,.sha256sum}
```

**Linux**
```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

### Install Cilium
You can install Cilium on any Kubernetes cluster. These are the generic instructions on how to install Cilium into any Kubernetes cluster. The installer will attempt to automatically pick the best configuration options for you.

#### Requirements
- Kubernetes must be configured to use CNI
- Linux kernel >= 4.9.17

#### Install
Install Cilium into the Kubernetes cluster pointed to by your current kubectl context:
```bash
cilium install
```

If the installation fails for some reason, run `cilium status` to retrieve the overall status of the Cilium deployment and inspect the logs of whatever pods are failing to be deployed.

### Validate the Installation
To validate that Cilium has been properly installed, you can run
```bash
$ cilium status --wait
   /¯¯\
/¯¯\__/¯¯\    Cilium:         OK
\__/¯¯\__/    Operator:       OK
/¯¯\__/¯¯\    Hubble:         disabled
\__/¯¯\__/    ClusterMesh:    disabled
   \__/

DaemonSet         cilium             Desired: 2, Ready: 2/2, Available: 2/2
Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Containers:       cilium-operator    Running: 2
                  cilium             Running: 2
Image versions    cilium             quay.io/cilium/cilium:v1.9.5: 2
                  cilium-operator    quay.io/cilium/operator-generic:v1.9.5: 2
```

Run the following command to validate that your cluster has proper network connectivity:
```bash
$ cilium connectivity test
ℹ️  Monitor aggregation detected, will skip some flow validation steps
✨ [k8s-cluster] Creating namespace for connectivity check...
(...)
---------------------------------------------------------------------------------------------------------------------
📋 Test Report
---------------------------------------------------------------------------------------------------------------------
✅ 69/69 tests successful (0 warnings)
```
Congratulations! You have a fully functional Kubernetes cluster with Cilium. 🎉