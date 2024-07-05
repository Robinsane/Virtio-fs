# Virtio-fs-Hookscript

Hookscript to create Virtio-fs mounts, to share a directory tree on a Proxmox host with VMs.

The source of the script is this [Proxmox Forums tutorial](https://forum.proxmox.com/threads/virtiofsd-in-pve-8-0-x.130531/) and it has been modified by [@dvino](https://gist.github.com/dvino) and [Drallas](https://github.com/Drallas) to be more flexible.

# Mount Volumes into Proxmox VMs with Virtio-fs

> Part of collection: [Hyper-converged Homelab with Proxmox](https://gist.github.com/Drallas/57c37dfea89be2d41134cc00d34503e4)

[Virtio-fs](https://virtio-fs.gitlab.io) is a shared file system that lets virtual machines access a directory tree on the host. Unlike existing approaches, it is designed to offer local file system semantics and performance. The new **virtiofsd-rs** Rust daemon Proxmox 8 uses, is receiving the most attention for new feature development. 

Performance is very good (while testing, almost the same as on the Proxmox host)

VM Migration is not possible yet, but it's being [worked on](https://gitlab.com/virtio-fs/virtiofsd/-/issues/81#note_3)!

## Architecture
<img width="1468" alt="Screenshot 2023-09-27 at 17 47 52" src="https://user-images.githubusercontent.com/24792888/271039506-87877f12-4c78-4827-b1e1-eca8c53c7a2d.png">


## Why?
Since I have a [Proxmox High Available cluster with Ceph](https://gist.github.com/Drallas/96fa494b84af7e30b68e1dc0d177812f), I like to mount the Ceph File System, with CephFS Posix-compliant directories into my VM’s. I have been playing around with LXC container and Bind Mounts and even successfully setup [Docker Swarm in LXC Containers](https://gist.github.com/Drallas/e03eb5a4f68bb526f920a423455bc0c9). Unfortunately, this is not a recommended configuration and comes with some trade-offs and cumbersome configuration settings.

This Write-Up explains how to [Create Erasure Coded CephFS Pools](https://gist.github.com/Drallas/6069c5fc96f3570a9c8d573fc9c6d44c) to store Volumes that than can be mounted into a VM via [virtiofs](https://gitlab.com/virtio-fs/virtiofsd).

---

## Install virtiofsd

| This procedure has been tested with Ubuntu Server 22.04 and Debian 12!

Proxmox 8 Nodes, don’t have **virtiofsd** installed by default, so the first step is to install it.

```bash
apt install virtiofsd -y

# Check the version
/usr/lib/kvm/virtiofsd --version
virtiofsd backend 1.7.0
```
> virtiofsd 1.7.0 has many issues (hangs after rebooting the vm, superblock errors etc...) version 1.7.2 and 1.8.0 seems to work much better, it can be found at [virtio-fs releases](https://gitlab.com/virtio-fs/virtiofsd/-/releases) page. But be carefull this package is not considered stable and not even in unstable [Debian Package Tracker](https://tracker.debian.org/pkg/rust-virtiofsd).

---

## Add the hookscript to the vm

Still on the Proxmox host!

Get the [Hookscript files](https://github.com/Drallas/Virtio-fs-Hookscript/tree/main/Script) files and copy them to `/var/lib/vz/snippets`, and make `virtiofs_hook.pl` executable.

Or use the [get-hookscript.sh](https://github.com/Drallas/Virtio-fs-Hookscript/blob/main/get_hook_script.sh) script to download the scripts files automatically to `/var/lib/vz/snippets`.

```bash
cd ~/
sudo sh -c "wget https://raw.githubusercontent.com/Drallas/Virtio-fs-Hookscript/main/get_hook_script.sh"
sudo chmod +x ~/get-hook%20script.sh
./get-hook%20script.sh
```

### Modify the conf file

To set the **VMID** and the **folders** that a VM needs to mount, open the virtiofs_hook.conf file.


```bash
sudo nano /var/lib/vz/snippets/virtiofs_hook.conf
```

### Set the Hookscript to an applicable VM

Set the hookscript to a VM.

```bash
qm set <vmid> --hookscript local:snippets/virtiofs_hook.pl
```

That's it, when it's added to the VM, the script does it magic on VM boot:

- Adding the correct Args section to the virtiofsd
`args: -object memory-backend-memfd,id=mem,size=4096M,share=on -numa node........`
- Creating the sockets that are needed for the folders.
- Cleanup on VM Shutdown

---

## Start / Stop VM

The VM can now be started and the hookscript takes care of the virtiofsd part.

```bash
qm start <vmid>
```

### Check

Check the processes virtiofsd `ps aux | grep virtiofsd` or `systemctl | grep virtiofsd` for the systemd services.

If all is good, it looks like this:
<img width="2308" alt="Screenshot 2023-09-23 at 12 51 55" src="https://user-images.githubusercontent.com/24792888/270102663-2f00351f-fbc1-46fd-82be-3e94bc50565b.png">

---

## Mount inside VM

**Linux kernel >5.4 inside the VM, supports Virtio-fs natively**

Mounting is in the format:
`mount -t virtiofs <tag> <local-mount-point> `

To find the **tag**

On the Proxmox host; Exucute `qm config <vmid> --current` and look for the `tag=xxx-docker` inside the args section `args: -object memory-backend-memfd,id=mem,size=4096M,share=on -numa node,memdev=mem -chardev socket,id=char1,path=/run/virtiofsd/xxx-docker.sock -device vhost-user-fs-pci,chardev=char1,tag=<vmid>-<appname>`

```bash
# Create a directory
sudo mkdir -p /srv/cephfs-mounts/<foldername>

# Mount the folder
sudo mount -t virtiofs mnt_pve_cephfs_multimedia /srv/cephfs-mounts/<foldername>

# Add them to /etc/fstab
sudo nano /etc/fstab

# Mounts for virtiofs
# The nofail option is used to prevent the system to hang if the mount fails!
<vmid>-<appname>  /srv/cephfs-mounts/<foldername>  virtiofs  defaults,nofail  0  0
 
# Mount everything from fstab
sudo systemctl daemon-reload && sudo mount -a

# Verify
ls -lah /srv/cephfs-mounts/<vmid>-<appname>
```

## Issues
1. New Vm's tend to trow a 'superblock' error on first boot:
```bash
mount: /srv/cephfs-mounts/download: wrong fs type, bad option, bad superblock on mnt_pve_cephfs_multimedia, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
```
To solve this, I poweroff the vm `sudo /sbin/shutdown -HP now` and then start it again from the host with `qm start <vmid>`, everything should mount fine now.

2. Adding an extra volume throws also a 'superblock' error. 
```bash
qm stop <vmid>
sudo nano /etc/pve/qemu-server/<vmid>.conf
# Remove the Arg entry
`args: -object memory-backend-memfd,id=mem,size=4096M,share=on..
qm start <vmid>
```
Now the Volume's all have a superblock error; I poweroff the vm `sudo /sbin/shutdown -HP now` and then start it again from the host with `qm start <vmid>`, everything should mount fine again.

---

## Cleanup

To remove Virtio-fs from a VM and from the host: 

`nano /etc/pve/qemu-server/xxx.conf`
```bash
# Remove the following lines
hookscript: local:snippets/virtiofs-hook.pl
args: -object memory-backend-memfd,id=mem,size=4096M,share=on..
```

Disable each virtiofsd-xxx service, replace xxx with correct values or use (* wildcard) to remove them all at once.
```bash
systemctl disable virtiofsd-xxx
sudo systemctl reset-failed virtiofsd-xxx
```

This should be enough, but if the reference persist:
```bash
# Remove leftover sockets and services.
rm -rf /etc/systemd/system/virtiofsd-xxx
rm -rf /etc/systemd/system/xxx.scope.requires/
rmdir /sys/fs/cgroup/system.slice/'system-virtiofsd\xxx' 
```

If needed reboot the Host, to make sure all references are purged from the system state.

---

## Links

- **[[TUTORIAL] virtiofsd in PVE 8.0.x](https://forum.proxmox.com/threads/virtiofsd-in-pve-8-0-x.130531/)**
- **[Sharing filesystems with virtiofs between multiple VMs](https://blog.domainmess.org/post/virtiofs/)**
- **[virtiofsd - vhost-user virtio-fs device backend written in Rust](https://gitlab.com/virtio-fs/virtiofsd)**

---
