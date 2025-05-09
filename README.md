# Cluster configuration from the command line.

## Setting up the headnode, arachne.

**NOTE:** this only needs to be done once. This section documents the changes.


### Enable NAT

This allows the compute nodes to reach the Internet. 

`sudo sysctl -w net.ipv4.ip_forward=1`

`echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf`

```bash
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-source=10.0.0.0/24
sudo firewall-cmd --reload
```

### Enable the standard NFS protocols.

*NB:* Warewulf appears to use some other mechanism.

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind

sudo firewall-cmd --reload
```

### Setup /etc/exports

*NB:* We don't export `/usr/local` directly because CUDA will not function
correctly from a read-only share.

```bash
/home 10.0.0.0/24(rw,sync,no_root_squash,no_subtree_check,fsid=0)
/opt 10.0.0.0/24(ro,sync,no_root_squash,no_subtree_check,fsid=1)
```

## Configuring a compute node.

### Hardware attachment

- Use the tiny monitor. The VGA plug is on the backplane, near the center. There is only one video connection. 
- Use the black keyboard and mouse that share a dongle. The mouse is not strictly necessary. The USB plugs are on the backplane near the VGA plug.

### Before the install.

- Plug in the Rocky 9.5 USB.
- Reboot the node and hold down the DEL key to enter the SETUP menu.
- Go to the boot section. 
    - Make the USB stick the top priority boot option.
    - Delete the options having to do with PXE. There are four or five variations on this theme.
    - Save the changes, and reboot.

### Install Rocky 9.5 from USB

- Only configure the `root` user. Be sure to enable remote login with password for the `root` user; it is needed before the key-based access is available.
- Again, do not configure an additional user beyond root.
- Choose the simple server, and add the packages for NFS client, headless management, and text based server tools. 
- Use the default layout for storage. Delete all the paritions on both SSDs.
- Set the hostname as `nodeNN`, where NN is the node number.
- There is only one network that is plugged in, although each compute node has four NICs.
- In the network config, set the IP address to `10.0.0.NN`, netmask to `255.255.255.0`, and the gateway to be arachne's IP address, `10.0.0.254`
- Enable it for use by any user and to start on boot. 
- Click the "begin install" button. **NOTE:** The installation does not take very long.

### 
