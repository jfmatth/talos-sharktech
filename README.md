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

