# Talos on SharkTech

## Architecture

Sharktech 1x2, Ubuntu 25.10

``eth0`` = Public IP  
``eth1`` = Private VNET (192.168.50.0/24) 192.168.50.1 (DGW)

## IP Forwarding for DNAT to cluster
```
sudo nano /etc/sysctl.conf
```

Add 
```
net.ipv4.ip_forward=1
```

Apply
```
sudo sysctl -p
```

## DNAT for 80 and 443
```
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.50.10:80
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j DNAT --to-destination 192.168.50.10:443
```

**Add Talos later**

## Talos

### Control Plane
Provision a Talos Control Planeon the vNetLAS network

- 2X4 vm
- Talos 1.13.5 template
- Admin name / password don't matter
- No public IP
- Private network, 192.168.50.10

From the public VM...  

Following the instructions here - https://docs.siderolabs.com/talos/v1.13/getting-started/getting-started#step-3-store-your-node-ip-addresses-in-a-variable

```
export CONTROL_PLANE_IP=192.168.50.10
export CLUSTER_NAME=talos-sharktech
export DISK_NAME=sda
talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 \
    --install-disk /dev/$DISK_NAME \
    --config-patch @patch-static.yaml \
    --config-patch @patch-resolver.yaml \
    --config-patch @patch-hostname.yaml \
    --force
```

Apply the config to the running VM
```
talosctl apply-config --insecure \
    --nodes $CONTROL_PLANE_IP \
    --file controlplane.yaml
```

Set the endpoints
```
talosctl --talosconfig=./talosconfig config endpoints $CONTROL_PLANE_IP
```

Bootstrap
```
talosctl bootstrap --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
```

Watch the dashboard until all signs show "READY"
```
talosctl dashboard --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
```

Check health just to be sure
```
talosctl --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig health
```

Save all context stuff
```
talosctl config add talos-sharktech
cp talosconfig ~/.talos/config
talosctl config endpoint $CONTROL_PLANE_IP
talosctl config node $CONTROL_PLANE_IP
talosctl config contexts
```

## Talos Upgrades
Current Sharktech template is v1.13.5

Upgrade to 1.13.6 via https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/lifecycle-management/upgrading-talos#upgrade-api-changes-in-talos-v1-13

```
talosctl upgrade --nodes 192.168.50.10 --image ghcr.io/siderolabs/installer:v1.13.6
```