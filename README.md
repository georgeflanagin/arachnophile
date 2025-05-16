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

- Use the tiny monitor. The VGA plug is on the backplane of the CPU nodes and front panel of the GPU nodes, near the center. There is only one video connection. 
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
- Choose 8.8.8.8 for DNS
- Enable it for use by any user and to start on boot. 
- Click the "begin install" button. **NOTE:** The installation does not take very long compared to installs of a GUI-based Linux.

### After booting into Rocky 9.5

- It boots into text mode. Verify you can login with the password.
- Do a `dnf update` FIRST.
- Disable SELinux
    - `setenforce 0`
    - Edit `/etc/selinux/config`
        - Change the SELINUX line to say `SELINUX=disabled`.
- Now ensure you are seeing the outside world: 
    - `ping 10.0.0.254` should ping Arachne.
    - `ping 8.8.4.4` should work.
    - `dnf install epel-release` should work.
    - `dnf install cowsay` should work.
- Login to Arachne as root. 
    - `ssh-copy-id root@nodeNN`
    - Now login to the node from Arachne
- Remove the directories that we need for mounts. (They should be empty... If they are not, proceed with `rm -fr`.)
    - `rmdir /opt`
    - `mkdir /opt`
    - `umount /home`
    - `rmdir /home`
    - `mkdir /home`
- Prevent accidental updates to critical packages.
    - add this line to `/etc/dnf/dnf.conf`:
        - `exclude=kernel* kmod-kvdo* zfs*`
- Edit `/etc/fstab`
    - Delete the line that references `/home`
    - Add two lines to reference `/home` and `/opt` as mounts on Arachne.
```bash
10.0.0.254:/home /home nfs defaults,_netdev 0 0
10.0.0.254:/opt  /opt nfs ro,_netdev 0 0
```
- Mount the shared disks from Arachne.
    - `mount -av` (the -v is verbose so that errors will be detailed instead of only reported.)
    - `ls -l /home` (Should show users' `$HOME`s from Arachne.)
- Add these lines to `/etc/hosts` so that the nodes "know about" each other
```
10.0.0.254  arachne
10.0.0.1    node01
10.0.0.2    node02
10.0.0.3    node03
10.0.0.51   node51
10.0.0.52   node52
10.0.0.53   node53
```
- Create a link to our research software (Note that this step is a convenience so that Arachne resembles the other cluster computers at UR.) 
    - `cd /usr/local`
    - `ln -s /opt/sw/pub/apps sw`
    - `cd sw` and `ls -l` to verify.
- Enable the NVIDIA repo (if the node has GPUs)
    - `sudo curl -o /etc/yum.repos.d/cuda-rhel9.repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo`
    - verify it got done: `dnf repolist`
- Install one or more CUDA libraries
    - `dnf install cuda` (whatever is most recent and still compatible with the OS)
    - `dnf install cuda-12-1` As an example.
    - These will be installed into canonical paths on `/usr/local`.

- Install SLURM
    - `dnf install slurm slurm-slurmd slurm-slurmctld slurm-perlapi slurm-torque munge`  ( **NOTE** the version of slurm must be the same throughout the cluster.)
    - Create the munge key on Arachne (This is already done, but it seems like a good idea to document it here.) 
```bash
    dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key
    chown munge:munge /etc/munge/munge.key
    chmod 400 /etc/munge/munge.key
```
    - From Arachne, copy the munge key to the node.
```bash
    scp /etc/munge/munge.key root@nodeNN:/etc/munge/
    ssh root@nodeNN "chown munge:munge /etc/munge/munge.key && chmod 400 /etc/munge/munge.key"
```
    - Get munge running: `systemctl enable --now munge`
    - Create directories for SLURM's bookkeeping on the node. SLURM is very picky about owners and permissions.
        - `mkdir -p /var/spool/slurmd /var/log/slurm`
        - `chown slurm:slurm /var/spool/slurmd /var/log/slurm`
    - Copy the slurm config files from Arachne to the node.
        - `scp /etc/slurm/slurm.conf nodeNN:/etc/slurm`
        - `scp /etc/slurm/prolog.sh nodeNN:/etc/slurm`
        - `scp /etc/slurm/slurm.epilog.clean nodeNN:/etc/slurm`
