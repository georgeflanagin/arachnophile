# Cluster configuration from the command line.

## Setting up the headnode, arachne.

### Enable NAT

This allows the compute nodes to reach the Internet. 

`sudo sysctl -w net.ipv4.ip_forward=1`

`echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf`

```bash
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-source=10.0.0.0/24
sudo firewall-cmd --reload
```


## Configuring a compute node.

### Install Rocky 9.5 from USB

- Choose the simple server, and add the packages for NFS client, headless management, and text based server tools. 
- Use the default layout for storage.
- Set the hostname as `nodeNN`, where NN is the node number.
- There is only one network that is plugged in, although each compute node has four NICs.
- In the network config, set the IP address to `10.0.0.NN`, netmask to `255.255.255.0`, and the gateway to be arachne's IP address, `10.0.0.254`
- Enable it for use by any user and to start on boot. .
