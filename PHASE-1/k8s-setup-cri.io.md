Step 1: Enable iptables Bridged Traffic on all the Nodes

Execute the following commands on all the nodes for IPtables to see bridged traffic. Here we are tweaking some kernel parameters and setting them using sysctl.

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system without reboot
```
