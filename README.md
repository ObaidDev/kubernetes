# ``k3s Cluster Setup on DigitalOcean with VPC Networking``



## Overview
This guide explains how to deploy a **k3s cluster** on DigitalOcean **across multiple droplets** while ensuring **secure pod-to-pod communication** within a **VPC**.


## Prerequisites

Before setting up the k3s cluster, please follow the [k3sup installation guide](https://blog.alexellis.io/create-a-3-node-k3s-cluster-with-k3sup-digitalocean/) to install and configure k3s on your droplets.

**Note**: Although the guide is tailored for DigitalOcean, the setup steps are also applicable to other VPS providers, such as AWS, Linode, and Vultr, as long as the droplets are connected to the same private network (VPC) and have public IPs.

Alternatively, if you prefer not to deal with installation outside your VPC network, you can follow the official [K3s quick-start guide](https://docs.k3s.io/quick-start) for a simpler installation process.

Once you've completed the setup, you can proceed with the remaining steps for configuring pod-to-pod communication within the VPC.


The guide includes:
- Installing k3sup
- Setting up the master node
- Adding worker nodes
- Testing the k3s installation

## Problem
The nodes need to be able to reach other nodes over UDP port 8472 when using the **Flannel VXLAN backend , or over UDP port 51820 (and 51821 if IPv6 is used) when using the Flannel WireGuard backend**. The node should not listen on any other port. K3s uses reverse tunneling such that the nodes make outbound connections to the server and all kubelet traffic runs through that tunnel.

Check This Section For More Details [ K3s Doc ](https://docs.k3s.io/installation/requirements#networking).



## Solution: Configure Flannel to Use the Private Network
To force Flannel to use the **private VPC network**, we need to:
1. **Open UDP port 8472** for internal traffic **within the VPC**.
2. **Ensure k3s uses the private network interface** (`eth1` or equivalent).



---

## **Step 1: Allow VXLAN Traffic in DigitalOcean Firewall**
- Create an **inbound rule** for **all nodes**:
  - **Protocol**: UDP  
  - **Port**: `8472`  
  - **Source**: **VPC CIDR block** (e.g., `10.0.0.0/16`)  

- Also allow:
  - **UDP 51820** (if using WireGuard backend)
  - **TCP 6443** (for agent nodes to connect to the master)

---

## **Step 2: Find the Private Network Interface**
Run the following command on any droplet:

```sh
ip a | grep 'inet 10'
```

__Example output :__
```
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 10.123.45.67/24 brd 10.123.45.255 scope global eth1
```
**The private interface is eth1 (it may be ens3 or another name).**

---

## **Step 3: Update k3s Configuration**
- On the Server Node Edit the k3s service file:
  - `sudo nano /etc/systemd/system/k3s.service`  
  - **Find the ExecStart line and update it**: `ExecStart=/usr/local/bin/k3s server --node-ip=10.X.X.X --flannel-iface=eth1`  
  - **Replace 10.X.X.X with the private IP of the server node.** 

- On the Agent Nodes Edit the agent service file :
  - `sudo nano /etc/systemd/system/k3s-agent.service`  
  - **Find the ExecStart line and update it**: `ExecStart=/usr/local/bin/k3s agent --server https://10.X.X.X:6443 --token your-token --node-ip=10.Y.Y.Y --flannel-iface=eth1`  
  - **10.X.X.X → Private IP of the server node**
  - **10.Y.Y.Y → Private IP of the agent node**



## **Step 4: Apply the Changes**
- Reload systemd and restart k3s :
  - `sudo systemctl daemon-reload`  
  - `sudo systemctl restart k3s`  
  - **For agents:** `sudo systemctl restart k3s-agent`



## **Step 5: Verify Connectivity**
- Create Dummy Pods And Try to Do Pod To Pod communication


---

## By

* Author: Obaid
* Date: 2025-03-11