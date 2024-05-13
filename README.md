***
# Description

The purpose of this write-up is to document how I configured storage in my NAS. 
For this setup, I've opted for a lightweight Linux _Container_ (LXC) to share the host's _ZFS_ file system. This will enable me to easily manage shared storage and user access via a web-based interface using _Cockpit_, along with additional modules from _45Drives_ that integrate with Samba. 

The ZFS file system is known for its robust features particularly in enterprise environments, including:

- **Deduplication:**  The elimination of redundant data copies to optimise storage usage.
- **Compression:** The reduction in the number of bits required to represent data, leading to increased storage efficiency.
- **Snapshots:** Point-in-time references that track changes to data over time.
- **Cloning:** The creation of identical duplicates for rapid data replication and backup purposes.
- **Data redundancy:** Advanced safeguards ensuring the integrity and security of critical information.

The primary disadvantage of using a ZFS mirror is its increased storage requirements; it requires twice as many physical disks to store the same amount of data compared to other VDEV types. With my two 12TB drives, my total working capacity will be 12TB, as the other disk is used for parity. 

***
## Environment
**Proxmox 8.1.10**

- LXC container: debian-12-standard_12.2-1
  - Memory: 512MB
  - Cores: 2
  - Networking: static IP 10.0.0.2/24
  
Storage
  - 2x **Seagate IronWolf 12TB NAS Internal Hard Drive 3.5 inch**
    - Configured in a ZFS RAID z1 pool on Proxmox


***Create ZFS zpool***

![Here I create a zpool and name it:star2: tank :star2:](https://i.imgur.com/eNdU8Ys.png)

***
### Container Set-up

![Container Creation](https://i.imgur.com/VEMgAOW.png)

After starting the Debian LXC container, in the shell, run the following commands to upgrade to latest:
```
apt update
apt upgrade
```

Install cockpit:
```
apt install --no-install-recommends cockpit -y
```

After installing Cockpit, we need to temporarily allow root access for setup purposes. To do so, edit the `/etc/cockpit/disallowed-users` file and comment out the `root` entry. 

```
nano /etc/cockpit/disallowed-users
```


Once this modification is made, you can connect to Cockpit at `https://10.0.0.3:9090` using your root credentials, and it should function as expected.

![Cockpit Interface](https://i.imgur.com/1rJFsIr.png) 


**After creating users and their permissions, add `root` back into the `/etc/cockpit/disallowed-users` file for security purposes.**

  ```
GNU nano 7.2            /etc/cockpit/disallowed-users                     
# List of users which are not allowed to login to Cockpit
root
```

***
### Install the three modules from 45Drives: 
 
> [!NOTE] 
> Before proceeding, be sure to check for the latest releases of each `.deb` package
 
- [45Drives Cockpit File Sharing](https://github.com/45Drives/cockpit-file-sharing)
- [45Drives Cockpit Navigator](https://github.com/45Drives/cockpit-navigator)
- [45Drives Cockpit Identities](https://github.com/45Drives/cockpit-identities)

![Github release versions](https://i.imgur.com/SuIDQVF.png)

cd to `/root`


wget each of the three 45Drives modules

Cockpit File Sharing
```
wget https://github.com/45Drives/cockpit-file-sharing/releases/download/v3.3.6/cockpit-file-sharing_3.3.6-1focal_all.deb
```

Cockpit Navigator
```
wget https://github.com/45Drives/cockpit-navigator/releases/download/v0.5.10/cockpit-navigator_0.5.10-1focal_all.deb
```

Cockpit Identities
```
wget https://github.com/45Drives/cockpit-identities/releases/download/v0.1.12/cockpit-identities_0.1.12-1focal_all.deb
```

Install them:
```
apt install ./*.deb

#It will complain about being unable to delete the deb files, so we will do that now
rm ./*.deb
```
***
#### Add a mount point
After creating the container and setting up Cockpit, it is time to add a mount point in the containers resources on Proxmox. This will point the container to the zpool (tank) we created.

This is one of the benefits of using a LXC container and Cockpit to share out storage on the Proxmox underlying ZFS backed storage, as the mount points, and container resources can be modified on the fly while CT is running. 

- Storage: _tank_
- Path: _/mnt/rust_
- Size: _10000 GiB_  (~10TB)

![Proxmox GUI - adding Mount Point](https://i.imgur.com/Oko8L3e.png)


![Proxmox GUI - Mount Point settings](https://i.imgur.com/BX5g2Ik.png)

**An error I encountered when creating the Mount Point**
_Fixed by restarting container and re-adding Mount Point._	
![Proxmox GUI error](https://i.imgur.com/vdUi6jm.png)


After successfully adding the mount point in Proxmox, the web-ui for Cockpit shows the mount point that we created. 

![/mnt/rust](https://i.imgur.com/1AxCRHI.png)

***
#### Creating Users and Groups

To manage access to the storage drive, we can use ACL's with both a user group and user identity.

a group that defines who has permissions to access the storage.
users are individual users who can are going to be using the system and can be given different permissions depending what they need access to.

1. In the Cockpit web-ui, navigate to _Identities_ and then to _Groups_
2. Create a new group, I called mine `rust-users`
	![](https://i.imgur.com/f3uQUEL.png)

3. Navigate back to _Identities_ and create a new _User_ 
	- Set the new users Login Shell to `/bin/bash`
	- Add the new user to the `rust-users` group
	![](https://i.imgur.com/NY83C0a.png)

4. Set passwords
	- Windows employs a distinct hashing algorithm compared to Linux, necessitating the creation of a separate SMB (SAMBA) password.


***
#### Creating Samba Share and Permissions

Windows has its own distinct approach to access control, which is actually quite effective. NFS (Network File System) on Linux adopted Windows-style Access Control Lists (ACLs), but Linux itself did not.

As such, you'll typically want to use Windows ACLs. When you select this option, Samba will completely disregard the file permissions on the Linux system and manage permissions itself. For my use case, using Windows ACLs are effective. 

![](https://i.imgur.com/F1paCvU.png)

**Issue with copy to share from a Linux host (Fedora OS)** 
_Needed to change the group permissions to add write access for Group_

![](https://i.imgur.com/5Ah3Ru0.png)

### Conclusion

In this write-up, I have documented my configuration of storage in my NAS, utilising Linux Containers (LXC) to share the host's ZFS file system. 

By leveraging LXC and Cockpit, I am able manage shared storage and user access via a web-based interface. Additionally, with the integration of 45Drives modules and Samba, there is data sharing across the LAN. 

![Samba share mounted to Ubuntu Desktop host](https://i.imgur.com/x786H4M.png)


![Prompt for credentials when attempting to access share on a Windows 11 VM.](https://i.imgur.com/iPqBF8O.png)


### References
- [Turning Proxmox Into a Pretty Good NAS](https://www.youtube.com/watch?v=Hu3t8pcq8O0)
- [Single Server HomeLab Using Proxmox (Proxmox Hypervisor and NAS)](https://www.youtube.com/watch?v=mKfqQj9wUuE)


