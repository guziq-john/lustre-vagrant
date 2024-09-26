# Build Lustre from source code

1. Run `sudo vagrant up --provider=qemu` to start the CentOS 8 VM.
2. Run `sudo vagrant ssh` to get into the VM.
3. Run below commands to build lustre:

```bash
git clone https://github.com/openzfs/zfs.git

# Build
cd ~/zfs
git checkout -b coverity-run zfs-2.1.11
sh autogen.sh
./configure
make -s -j\$(nproc)

# Install
sudo make install

# Grab repo
git clone git://git.whamcloud.com/fs/lustre-release.git

# Build
cd ~/lustre-release
./autogen.sh
./configure
make -s -j\$(nproc)

# Report
echo "KERNEL MODULES BUILT:"
find . -name *.ko
EOF
```