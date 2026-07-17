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
    --config-patch-control-plane @cp-patch-network.yaml \
    --force
```

Apply the config to the running VM
```
talosctl apply-config --insecure \
    --nodes $CONTROL_PLANE_IP \
    --file controlplane.yaml
```

Save all context stuff
```
talosctl config add talos-sharktech
cp talosconfig ~/.talos/config
talosctl config endpoint $CONTROL_PLANE_IP
talosctl config node $CONTROL_PLANE_IP
talosctl config contexts
talosctl kubeconfig -f ~/.kube/config
```


Endpoints / bootstrap / dashboard / healthcheck
```
talosctl --talosconfig=./talosconfig config endpoints $CONTROL_PLANE_IP
talosctl bootstrap --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
talosctl dashboard --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
talosctl --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig health
```

Set Hostname
```
talosctl patch machineconfig --talosconfig=./talosconfig --nodes $CONTROL_PLANE_IP -p @cp-patch-hostname.yaml
```

<!-- Fix DNS - Use host DNS instead of ???
```
talosctl patch machineconfig -n 192.168.50.10 --patch @dns-fix-patch.yaml
``` -->

Cillium
```
helm install cilium cilium/cilium --namespace kube-system -f cilium.yaml --version 1.18.9
kubectl apply -f cilium-announce.yaml
```


### Traefik
https://docs.siderolabs.com/kubernetes-guides/advanced-guides/deploy-traefik#deploy-traefik-as-a-gateway-api

```
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
kubectl apply -f traefik-namespace.yaml
helm install traefik traefik/traefik -f traefik-values.yaml -n traefik
kubectl apply -f traefik-gateway.yaml
```

### Nodes

Node 1
```
talosctl apply-config --insecure --nodes 192.168.50.11 --file worker.yaml
talosctl patch machineconfig --talosconfig=./talosconfig --nodes 192.168.50.11 -p @nd1-patch-network.yaml
talosctl patch machineconfig --talosconfig=./talosconfig --nodes 192.168.50.11 -p @nd1-patch-hostname.yaml
```

## Talos Upgrades
Current Sharktech template is v1.13.5

Upgrade to 1.13.6 via https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/lifecycle-management/upgrading-talos#upgrade-api-changes-in-talos-v1-13

```
talosctl upgrade --nodes 192.168.50.10 --reboot-mode force
```