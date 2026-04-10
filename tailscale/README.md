apk add curl
apk add ethtool

ON THE PVE HOST
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/add-tailscale-lxc.sh)"

Back to LXC

```shell
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

```shell
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off
tailscale up --advertise-routes=192.168.1.0/24
```

This will not persist after reboots, so you have to run:

rc-update add sysctl boot


Helpful reference:
https://blog.xghozt.com/how-to-install-tailscale-on-proxmox-on-a-ct-with-a-subnet-router/
