---
title: "Site to Site VPN for Google Kubernetes Engine"
date: 2021-05-06T19:20:27+02:00
tags: ["kubernetes", "openvpn", "google"]
draft: false
---
In this tutorial I will try to explain you briefly and concisely how you can set up a site-to-site VPN for the Google Cloud Network.

### Prerequisites

We need 2 virtual machines. The first one on the side of our office and the other one on the side of Google.

#### Setup OpenVPN Clients

##### Site-to-Site Client Office Side
We need to install OpenVPN, we do it as follows:
```bash
apt install openvpn -y
```
After that we add our OpenVPN configuration under this path `/etc/openvpn/s2s.conf`.

*s2s.conf*
```
# Use a dynamic tun device.
# For Linux 2.2 or non-Linux OSes,
# you may want to use an explicit
# unit number such as "tun1".
# OpenVPN also supports virtual
# ethernet "tap" devices.
dev tun

# Our OpenVPN peer is the Google gateway.
remote IP_GOOGLE_VPN_CLIENT 

ifconfig 4.1.0.2 4.1.0.1

route 10.156.0.0 255.255.240.0            # Google Cloud VM Network
route 10.24.0.0 255.252.0.0               # Google Kubernetes Pod Network

push "route 192.168.10.0 255.255.255.0"   # Office Network 

# Our pre-shared static key
#secret static.key

# Cipher to use
cipher AES-256-CBC

port 1195

user nobody
group nogroup

# Uncomment this section for a more reliable detection when a system
# loses its connection.  For example, dial-ups or laptops that
# travel to other locations.
 ping 15
 ping-restart 45
 ping-timer-rem
 persist-tun
 persist-key

# Verbosity level.
# 0 -- quiet except for fatal errors.
# 1 -- mostly quiet, but display non-fatal network errors.
# 3 -- medium output, good for normal operation.
# 9 -- verbose, good for troubleshooting
verb 3

log /etc/openvpn/s2s.log
```
We also have to enable the IPv4 forward function in the kernel, so we go to `/etc/sysctl.conf` and comment out the following line:
```
net.ipv4.ip_forward=1
```
We can then start our OpenVPN client with this command:
```bash
systemctl start openvpn@s2s
```

On the Office side we have to open the port for the OpenVPN client that the other side can connect.

##### Site-to-Site Client Google Side
When setting up the OpenVPN client on Google's site, we need to consider the following settings when creating it. When we create the machine, we need to enable this option in the network settings:

![Google Cloud Network Settings](https://i.imgur.com/OXEkhxo.png)

Also on this side we have to install the OpenVPN client again and then add this config under the path `/etc/openvpn/s2s.conf`:
```
# Use a dynamic tun device.
# For Linux 2.2 or non-Linux OSes,
# you may want to use an explicit
# unit number such as "tun1".
# OpenVPN also supports virtual
# ethernet "tap" devices.
dev tun

# Our OpenVPN peer is the Office gateway.
remote IP_OFFICE_VPN_CLIENT 

ifconfig 4.1.0.2 4.1.0.1

route 192.168.10.0 255.255.255.0          # Office Network

push "route 10.156.0.0 255.255.240.0"     # Google Cloud VM Network
push "route 10.24.0.0 255.252.0.0"        # Google Kubernetes Pod Network

# Our pre-shared static key
#secret static.key

# Cipher to use
cipher AES-256-CBC

port 1195

user nobody
group nogroup

# Uncomment this section for a more reliable detection when a system
# loses its connection.  For example, dial-ups or laptops that
# travel to other locations.
 ping 15
 ping-restart 45
 ping-timer-rem
 persist-tun
 persist-key

# Verbosity level.
# 0 -- quiet except for fatal errors.
# 1 -- mostly quiet, but display non-fatal network errors.
# 3 -- medium output, good for normal operation.
# 9 -- verbose, good for troubleshooting
verb 3

log /etc/openvpn/s2s.log
```
We also have to enable the IPv4 forward function in the kernel, so we go to `/etc/sysctl.conf` and comment out the following line:
```
net.ipv4.ip_forward=1
```

##### Connection test
Now that both clients are basically configured we can test the connection. Both clients have to be started with systemctl. After that we look at the logs with `tail -f /etc/openvpn/s2s-log` and wait for this message:

```
Wed May  5 08:28:01 2021 /sbin/ip route add 10.28.0.0/20 via 4.1.0.1
Wed May  5 08:28:01 2021 TCP/UDP: Preserving recently used remote address: [AF_INET]0.0.0.0:1195
Wed May  5 08:28:01 2021 Socket Buffers: R=[212992->212992] S=[212992->212992]
Wed May  5 08:28:01 2021 UDP link local (bound): [AF_INET][undef]:1195
Wed May  5 08:28:01 2021 UDP link remote: [AF_INET]0.0.0.0:1195
Wed May  5 08:28:01 2021 GID set to nogroup
Wed May  5 08:28:01 2021 UID set to nobody
Wed May  5 08:28:11 2021 Peer Connection Initiated with [AF_INET]0.0.0.0:1195
Wed May  5 08:28:12 2021 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Wed May  5 08:28:12 2021 Initialization Sequence Completed
```

If we can't establish a connection, we need to check if the ports are opened on both sides.

#### Routing Google Cloud Network
After our clients have finished installing and configuring, we need to set the routes on Google. I will not map the Office side, as this is always different. But you have to route the networks for the Google network there as well.

To set the route on Google we go to the network settings and then to Routes. Here you have to specify your office network so that the clients in the Google network know what to do.

![Google Cloud Network Route](https://i.imgur.com/6Q2Drf4.png)

#### IP-Masquerade-Agent
IP masquerading is a form of network address translation (NAT) used to perform many-to-one IP address translations, which allows multiple clients to access a destination using a single IP address. A GKE cluster uses IP masquerading so that destinations outside of the cluster only receive packets from node IP addresses instead of Pod IP addresses. This is useful in environments that expect to only receive packets from node IP addresses.

You have to edit the ip-masq-agent and this configuration is responsible for letting the pods inside the nodes, reach other parts of the GCP VPC Network, more specifically the VPN. So, it allows pods to communicate with the devices that are accessible through the VPN.

First of all we're gonna be working inside the kube-system namespace, and we're gonna put the configmap that configures our ip-masq-agent, put this in a config file:

```yaml
nonMasqueradeCIDRs:
  - 10.24.0.0/14  # The IPv4 CIDR the cluster is using for Pods (required)
  - 10.156.0.0/20 # The IPv4 CIDR of the subnetwork the cluster is using for Nodes (optional, works without but I guess its better with it)
masqLinkLocal: false
resyncInterval: 60s
```
and run `kubectl create configmap ip-masq-agent --from-file config --namespace kube-system`

afterwards, configure the ip-masq-agent, put this in a `ip-masq-agent.yml` file:

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: ip-masq-agent
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        k8s-app: ip-masq-agent
    spec:
      hostNetwork: true
      containers:
      - name: ip-masq-agent
        image: gcr.io/google-containers/ip-masq-agent-amd64:v2.4.1
        args:
            - --masq-chain=IP-MASQ
            # To non-masquerade reserved IP ranges by default, uncomment the line below.
            # - --nomasq-all-reserved-ranges
        securityContext:
          privileged: true
        volumeMounts:
          - name: config
            mountPath: /etc/config
      volumes:
        - name: config
          configMap:
            # Note this ConfigMap must be created in the same namespace as the daemon pods - this spec uses kube-system
            name: ip-masq-agent
            optional: true
            items:
              # The daemon looks for its config in a YAML file at /etc/config/ip-masq-agent
              - key: config
                path: ip-masq-agent
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
      - key: "CriticalAddonsOnly"
        operator: "Exists"
```

and run `kubectl -n kube-system apply -f ip-masq-agent.yml`.

Now our site-to-site VPN should be set up. You should now test if you can ping the pods and if all other services work as you expect them to.