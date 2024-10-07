# lustre-vagrant

Provide an easy way to setup Lustre cluster with Vagrant and QEMU on MacOS with Apple M series CPU. The Lustre cluster has a server VM and a client VM. The MGS, MDS and OSS are all running in the single server VM.

## Preperations

Run below commands to install Vagrant, Libvirt and QEMU.

```bash
brew install hashicorp/tap/hashicorp-vagrant
brew install qemu libvirt virt-manager
vagrant plugin install vagrant-qemu
brew services start libvirt
```

## Lustre VMs

1. Clone this repository
2. Get into the repo folder, and create a disk file with below command:

    ```
    qemu-img create server-disk1.img 20G
    ```

3. Run `sudo vagrant up --provider=qemu` to boot up the two VMs, the server VM has a 20G raw disk.
4. Run `sudo vagrant status` to check the VM status

## VMs Preperations

1. Run `sudo vagrant ssh server` to get into the VM, and run `sudo -i` to switch to root user.
2. Run below commands to config the VM

    ```
    $ subscription-manager register  # you must have a RedHat developer account, which is free.
    $ systemctl disable firewalld
    $ systemctl stop firewalld
    $ systemctl mask firewalld
    ```

3. Edit `/etc/selinux/config`, and set `SELINUX=disabled` to disable seLinux.

4. Change the hostname with `hostname server && echo server > /etc/hostname`

5. Run `ifconfig eth1` to get the IP of eth1 and add `<eth1 IP> server` into `/etc/hosts`

6. Create file `/etc/modprobe.d/lustre.conf` with below content:

    ```
    options lnet networks="tcp0(eth1)"
    ```

4. Exit the VM terminal and run `sudo vagrant halt server` and then `sudo vagrant reload server` to reboot the VM.

Replace `server` to `client` in the above steps to config the client VM.

## Lustre Server Installation

After the server VM is rebooted, run `sudo vagrant ssh server`  and `sudo -i` to get into the VM.

1. Follow the section `Install ZFS packages` in [blog](https://metebalci.com/blog/lustre-2.15.4-on-rhel-8.9-and-ubuntu-22.04/) to install ZFS.
    Install ZFS packages
    RHEL does not by default support DKMS. epel-release package is required for that but it is not available from RedHat repositories. Install epel-release from Fedora project:
    $ subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
    $ dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    Then, add the ZFS repository:

    $ dnf install https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
    Then, install kernel-devel and zfs in DKMS style packages.

    $ dnf install kernel-devel
    $ dnf install zfs

3. Follow the section `Install Lustre servers with ZFS support` in [blog](https://metebalci.com/blog/lustre-2.15.4-on-rhel-8.9-and-ubuntu-22.04/) to install Lustre.

    Note: because the RHEL version installed in the VMs are 8.8, please use below Lustre yum repo config to install lustre-2.15.3.

    ```
    [root@server ~]# cat /etc/yum.repos.d/lustre.repo
    [lustre-server]
    name=lustre-server
    baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.3/el8.8/server/
    exclude=*debuginfo*
    enabled=0
    gpgcheck=0
    ```

4. Run below commands create the filesystem. Take the section `Configure Lustre file systems` in [blog](https://metebalci.com/blog/lustre-2.15.4-on-rhel-8.9-and-ubuntu-22.04/) as a reference, the main difference is that the MDT and OST have smaller size, becuase the disk size in the VM is only 20G. Download the `lustre-utils.sh` from [here](https://raw.githubusercontent.com/metebalci/lustre-utils.sh/refs/heads/main/lustre-utils.sh)

    ```
    $ ./lustre-utils.sh create_vg lustre /dev/vdb

    $ ./lustre-utils.sh create_mgt zfs

    $ ./lustre-utils.sh create_fs users zfs 1 1 zfs 3 2

    $ ./lustre-utils.sh create_fs projects zfs 1 1 zfs 2 2

    $ ./lustre-utils.sh start_mgs

    $ ./lustre-utils.sh start_fs users

    $ ./lustre-utils.sh start_fs projects

    $ ./lustre-utils.sh status
    ```

## Lustre Client Installation

Run `sudo vagrant ssh client`  and `sudo -i` to get into the VM.

1. Enable `codeready-builder` yum repo with below commands:

    ```
    $ subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
    $ dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    ````

2. Create lustre yum repo on file `/etc/yum.repos.d/lustre.repo` with below content:

    ```
    [lustre-client]
    name=lustre-client
    baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.3/el8.8/client/
    exclude=*debuginfo*
    enabled=0
    gpgcheck=0
    ```

3. Run `dnf --enablerepo=lustre-client install lustre-client-dkms lustre-client` to install lustre.

4. Run `lsmod | grep lustre` to check the installation, if there is no lustre modules, try `modprobe lustre`

5. Add `<the IP of eth1 on Server VM> server` to `/etc/hosts`

## Mount FS on Client

Run `sudo vagrant ssh client`  and `sudo -i` to get into the VM.

1. Run `modprobe lustre` to load the Lustre kernel modules
2. Make the mount point folder with `mkdir /users`
3. Run `mount -t lustre server:/users /users` to mount the `users` filesystem to `/users` folder
4. Verify lustre filesystem:

    ```bash
    # cd /users

    # lfs df -h
    UUID                       bytes        Used   Available Use% Mounted on
    users-MDT0000_UUID        815.8M        3.1M      810.6M   1% /users[MDT:0]
    users-OST0000_UUID          2.6G      660.0M        2.0G  25% /users[OST:0]
    users-OST0001_UUID          2.6G        3.0M        2.6G   1% /users[OST:1]

    filesystem_summary:         5.2G      663.0M        4.6G  13% /users

    # truncate -s 100M test
    # lfs getstripe test
    test
    lmm_stripe_count:  1
    lmm_stripe_size:   1048576
    lmm_pattern:       raid0
    lmm_layout_gen:    0
    lmm_stripe_offset: 1
        obdidx		 objid		 objid		 group
            1	        4	        0x4	         0
    ```
